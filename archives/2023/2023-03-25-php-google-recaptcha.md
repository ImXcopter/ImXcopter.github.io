# How to Integrate Google reCAPTCHA with PHP

在本教程中，我们将看到如何在 PHP 中使用 reCAPTCHA 或如何在 PHP 中使用 Google reCAPTCHA 代码。

## Step 1: Register Your Site and Get API Key

First, you need to register your website at Google reCAPTCHA admin console and get the site key and secret key.

**配置说明：**
- Label: name of your site
- reCAPTCHA type: Choose reCAPTCHA v2 >> Choose "I'm not a robot" Checkbox.
- Domains: Mention the domain name of your website.

![Register site to Google reCAPTCHA](/static/2023/2023-03-25-php-google-recaptcha_001.jpg)

Once submitted Google will provide you following two things:
1. Site Key
2. Secret Key

![Register site to Google reCAPTCHA](/static/2023/2023-03-25-php-google-recaptcha_002.jpg)

Copy the Google reCAPTCHA site key and secret key for later use in the reCAPTCHA integration code.

## HTML: Adding Google reCAPTCHA to Form

First, include the reCAPTCHA JavaScript API library. Paste this snippet before the closing head tag on your HTML template.
