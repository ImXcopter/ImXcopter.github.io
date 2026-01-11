# How to Integrate Google reCAPTCHA with PHP

在本教程中，我们将看到如何在PHP中使用Recaptcha或如何在PHP中使用Google reCaptcha代码。

Step 1. Register Your Site and Get API Key (Site Key and Secret key)
--------------------------------------------------------------------

First, you need to register your website at Google reCaptcha admin console and get the site key and secret key.
Label: name of your site
reCatpcha type: Choose reCaptcha v2 >> Choose I’m not a robot Checkbox.
Domains: Mention the domain name of your website.
![Register site to Google reCaptcha](/static/2023/2023-03-25-php-google-recaptcha_001.jpg)
Once submitted Google will provide you following two things:
1.Site Key
2.Secret Key
![Register site to Google reCaptcha](/static/2023/2023-03-25-php-google-recaptcha_002.jpg)
Copy the Google reCaptcha site key and secret key for later use in the reCaptcha integration code.

HTML [Adding Google reCaptcha to Form]
--------------------------------------

First, include the reCAPTCHA JavaScript API library. Paste this snippet before the closing head tag on your HTML template: