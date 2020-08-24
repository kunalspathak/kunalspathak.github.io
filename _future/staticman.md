disqus but not rendering
went with static man
- deployed heroku
- clone staticman repo
- configuration.production.json
- RSA/privatekey from https://yasoob.me/posts/running_staticman_on_static_hugo_blog_with_nested_comments/
- error of 
```
2020-06-28T06:26:57.141542+00:00 app[web.1]: Error [InvalidAsn1Error]: encoding too long
2020-06-28T06:26:57.141544+00:00 app[web.1]: at newInvalidAsn1Error (/app/node_modules/node-rsa/node_modules/asn1/lib/ber/errors.js:7:13)
2020-06-28T06:26:57.141544+00:00 app[web.1]: at Reader.readLength (/app/node_modules/node-rsa/node_modules/asn1/lib/ber/reader.js:102:13)
2020-06-28T06:26:57.141545+00:00 app[web.1]: at Reader.readSequence (/app/node_modules/node-rsa/node_modules/asn1/lib/ber/reader.js:135:16)
2020-06-28T06:26:57.141546+00:00 app[web.1]: at Object.privateImport (/app/node_modules/node-rsa/src/formats/pkcs1.js:63:16)
2020-06-28T06:26:57.141546+00:00 app[web.1]: at Object.detectAndImport (/app/node_modules/node-rsa/src/formats/formats.js:63:48)
2020-06-28T06:26:57.141547+00:00 app[web.1]: at NodeRSA.module.exports.NodeRSA.importKey (/app/node_modules/node-rsa/src/NodeRSA.js:185:22)
```

Then on printing inside RSA.js found out `cat key.pem` problem. Fixed it.
Some fixes needed for comments display logic which i copied from `https://github.com/Catch-up-TV-and-More/jekyll-website/blob/master/_includes/staticman-script.html`

1. Deploy heroku app
2. update _config.yml settings for staticman
3. Create blog-bot using https://travisdowns.github.io/blog/2020/02/05/now-with-comments.html
4. staticman forms
5. Test the comment
6. Remember to pull it from origin/master once PR is merged to see it on your local machine.