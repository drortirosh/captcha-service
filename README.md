# Captcha Service for GSN Paymaster

This project demonstrate how a paymaster (the on-chain contract that pays for the gasless transactions) 
can require an external service (in this case, CAPTCHA)
to approve a transaction.
This way, we can avoid spam from anonymous calling users.

https://www.google.com/recaptcha/admin/site/465756249/settings


The paymaster we use is a VerifyingPaymaster: it accepts any transaction, but only if it gets a signed approval
by the external service that the transaction is legit.

It is deployed as https://captcha-service.netlify.app/validate-captcha
For a sample project using it, please see https://github.com/opengsn/ctf-react/tree/captcha-paymaster

This project adds captcha check to paymaster requests

It involves 3 components:

1. **A CaptchaPaymaster**: a paymaster that requires a signed approvalData before it accepts a request.

2. **Captcha validation service**, which takes the captcha response from the webapp, validates it with google, and generates 
  a signed approvalData acceptable by the paymaster.

3. **The webapp**, which uses "Google Recaptcha", the validation service and the CaptchaPaymaster.


### Usage
1. The paymaster in this project accepts all requests. You might want to 
   inherit the CaptchaPaymaster, call `super.preRelayedCall()` and then add
   your restrictions.
 
2. register with Google's Captcha service https://www.google.com/recaptcha/admin/create
    - The example below uses reCaptcha v2 (with "I'm not a robot" button).
     
3. For testing purposes, you can deploy the validation service locally, by installing `netlify-cli`     

4. Deploy the validation server. This project is deployable directly to netlify.
   It is already deployed as gsn-captcha-paymaster.netlify.app
   
   Note that you need to provide 2 environment variables to the netlify app:
    - `RECAPTCHA_SECRET_KEY` - google's recaptcha secret key
    - `PRIVATE_KEY` - the private key that signs the approval 

   Without these items, the netlify deployment would fail. 

5. Go back to Google's reCaptcha admin page (https://www.google.com/recaptcha/admin), and update the netlify URL you deployed to
     (you can add `localhost` for testing)

6. Deploy the CaptchaPaymaster. Through truffle console, it can be deployed with:

        paymaster = await CaptchaPaymaster.new(signer,url, "captcha:")

    - `signer` - the address of the signer (for the above private key)
    - `url` of the the above validation service. e.g. 
            'https://gsn-captcha-paymaster.netlify.app/.netlify/functions/validate-captcha',
    - prefix for signing. currently has to be **"captcha:"**
    - make sure to connect it to the RelayHub and fund it.

7. Add "recaptcha" button to your webapp (see `html/index.html` for sample)
8. in you webapp, define your `RelayProvider` to use the above paymaster, and use
    the generated approval data:
    
    ```javascript
    import { createCaptchaAsyncApprovalCallback } from '@opengsn/captcha-paymaster'
    
    ...
    
    gsnProvider = new RelayProvider(provider, { ..., paymasterAddress: myCaptchaPaymasterAddress }, {
       asyncApprovalData: createCaptchaAsyncApprovalCallback(web3,
                               async() => grecaptcha.getResponse()) 
    })
    ```
   
9. Now transactions will only pass through if the user successfully proven he's not a robot...
        
### Technical Application flow

This is what happens under the hood:

1. The user has to click the "I'm not a robot" button. In the background, 
    a new captcha response is created.
2. When clicking the "Action" button, the RelayProvider attempts to create a request, and calls asyncApprovalData callback.
3. the callback calls a read function on the provider, to collect user's address, nonce, and current timestamp
4. Then the callback then calls the captcha validation service with the above user data.
5. The validation service uses Google API to validate the captcha, and signs the user's data and timestamp.
6. The RelayProvider can now use the returned "approvalData", and pass it with the next GSN request.
7. When processing this request, the paymaster validates the content is valid 
   (time not expired, user and nonce matches the request) and that the signature is valid.
8. The paymaster approves the request, and the transaction can be processed.

Note that in this sample, we explicitly don't attempt to refresh the captcha automatically:
If you dont press the "I'm not a robot" checkbox, a request will be sent with last captcha value,
which might be stale or completely invalid.
This comes to demonstrate that its not the client that validates the captcha, but
the paymaster itself.

