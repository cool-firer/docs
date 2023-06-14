

# 第5章 XSS

跨站脚本（Cross Site Scripting XSS）

原理：将恶意的HTML/JS代码注入Web网页，在未经有效安全过滤的情况下，在用户浏览器解释执行。

防范方案，在输入端和输出端：

1. 对用户输入部分进行过滤敏感词。

2. 白名单转义，如，将输出body中的关键词转义：

   ```html
   <  ---> &lt;
   >  ---> &gt;
   &  ---> &amp;
   "  ---> &quot;
   '  ---> &#39;
   ```

3. 设置HttpOnly属性，禁止脚本访问Cookie内容。





# 第6章 CSRF

跨站请求伪造（Cross Site Request Forgery）也称XSRF。

用户登录了A网站，此时浏览器保存有A网络的凭证（Cookie），此时访问恶意网站B，在B中有一个图片连接：

```html
// 构造Get、Post请求都可以。
<img src="http://A.com/transfer/100" />
```

浏览器会带上A的Cookie，发送后端，后端识别不出请求是否是用户发出的。

防范方案：

1. 后端检查请头部字段Origin、Referer，是否是A网站，如果不是，可能是被伪造了。
2. 每个非幂等的请求都带上一个CSRF Token。JWT是比较好的方案。



参考资料：https://tech.meituan.com/2018/10/11/fe-security-csrf.html









