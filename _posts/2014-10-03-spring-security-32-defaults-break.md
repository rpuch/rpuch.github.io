---
layout: post
title: Spring Security 3.2+ defaults break Wicket Ajax-based file uploads
date: '2014-10-03T22:31:00.000+04:00'
author: rpuchkovskiy
tags:
- Wicket
- Ajax file uploads broken
- Spring Security
- X-Frame-Options
modified_time: '2014-10-03T22:31:48.578+04:00'
blogger_id: tag:blogger.com,1999:blog-4160252863216482620.post-264219685757316513
blogger_orig_url: https://rpuchkovskiy.blogspot.com/2014/10/spring-security-32-defaults-break.html
---

A couple of days ago we have run into a bug: we found that file uploads in our
<a href="http://wicket.apache.org/" target="_blank">Wicket</a> application have broken. Instead of working as
expected, upload button did not work, instead a message appeared in the browser console (this one is for Chrome):

`Refused to display 'http://localhost:8084/paynet-ui/L7ExSNbPC4sb6TPJDblCAkN0baRJxw3q6-_dANoYsTD…QK61FV9bCONpyleIKW61suSWRondDQjTs8tjqJJOpCEaXXCL_A%2FL7E59%2FTs858%2F9QS3a' in a frame because it set 'X-Frame-Options' to 'DENY'.``

That seemed strange, because `X-Frame-Options` relates to frames which we didn't use explicitly. But when a
*file upload** is made using **Ajax**, Wicket carries this out using an **implicit Frame**.

<a href="http://projects.spring.io/spring-security/" target="_blank">Spring Security</a> started adding this header
starting with version 3.2, so it was actually an upgrade to Spring Security 3.2 that broke file uploads.
To sort this out, it was sufficiently to change the `X-Frame-Options` value from `DENY` to `SAMEORIGIN` using
the following snippet in web security configuration (created using
<a href="http://docs.spring.io/spring-security/site/docs/3.2.x/reference/htmlsingle/#jc" target="_blank">@Configuration-based approach</a>):

```
http
    .headers()
        .contentTypeOptions()
        .xssProtection()
        .cacheControl()
        .httpStrictTransportSecurity()
        .addHeaderWriter(new XFrameOptionsHeaderWriter(XFrameOptionsHeaderWriter.XFrameOptionsMode.SAMEORIGIN))
```

File uploads work now, the quest is finished.