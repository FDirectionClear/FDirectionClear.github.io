---
layout: post
title: CSS与JS的装载与执行
date: 2019-09-19
tag: 性能优化
---

一、一个网站在浏览器端是如何渲染的？
------------------------------------

![](/images/posts/2019-09-19-XNYH-loading-and-execution-of-css-and-js/2439705c6fbdf1750a9aabffdf888a4d.png)

在我们输入域名之后，浏览器会向服务器请求资源，而HTML文件则是我们最先请求回来的资源。**在请求回来之后，HTML本身会由字节流转换为字符流。在浏览器中拿到的就是字符流。然后浏览器利用html
parser进行词法分析，将相应的词法分析成语法的token，然后对这些token会进行next
token的方式添加到DOM树中去**。词法分析的是由上到下的，也就是说从html-\> body -\>
...。

在html词法解析的时候遇到如link的时候，浏览器就会并发的向服务器（或者一个其他的服务器）请求响应的静态资源，比如CSS。然后就会对CSS进行解析，形成响应的CSS树（CSSOM），最后会和DOM树进行结合形成渲染树（render树）。值得注意的是，只有DOM树和CSSOM结合之后形成render树之后，浏览器才会经过Layout和Paint的过程将图像绘制在页面上。

如果静态资源是Js，那也同样会并发请求，无论是代码中内联的javascript还是外部的，javascript都会交给浏览器的内部引擎V8去进行执行。

HTML渲染过程中的一些特点
------------------------

### 顺序执行、并发加载。

>   DOM树的解析按从上到下的顺序的，遇到外部资源会并发加载资源。当然在这里需要注意的是，并发加载并不是可以同时加载无数资源的（谷歌浏览器可以对一个域名同时并发请求6个资源），并发加载数会受到浏览器的域名限制。对于这些静态资源，一般情况下我们都会同时设3到4个cdn域名来储存他们，尽可能的保证所有资源都尽可能的同时加载。

### 是否阻塞

1.  css加载是否会阻塞js的下载：理论上会，但是实际上现代浏览器会进行预扫描，所以不会。

2.  css的加载是否会阻塞后续js的执行：会

3.  css的加载是否会阻塞页面的渲染： 会

4.  css的加载是否会阻塞DOM树的解析：不会

5.  js的加载是否会阻塞后续js的执行和加载： 会

*TIP：*

*具体原因，我的另一篇文章中有写，这里不再赘述。*

<https://fdirectionclear.github.io/2019/07/XNYH-zuseguanxi/>

### 依赖关系

确定好依赖关系确实能够解决很多类似于页面样式闪动的问题，比如一个常见的情况：页面的html加载好了，给我们看到的可能是没有样式的页面，一小段时间之后，页面突然出现了样式。

这种问题大多是我们没有确定好依赖关系造成的，核心原因就是因为浏览器在初次渲染完页面的时候，浏览器还没有发现css的参与，此时css还没有开始加载，等发现css，并在加载并解析后，浏览器就会进行了重新的渲染。为了解决这个问题，我们可以把css都写在head标签里面，这样渲染就会等待head中引入的css加载完成并生成render
Tree之后进行页面渲染，这样渲染出的成果就是一定带有样式的。就不会出现样式闪动的问题。

对于js而言，就们可以通过使用script标签的async特性来放弃依赖关系的方式来让js可以并发加载。毕竟js文件已经异步加载，所以能够提升页面的加载。但是这做法也存在缺点，就是我们必须让每个js有一定的独立性，相互不能进行依赖。如果需要依赖，可以使用defer。

1.  **引入方式**  
    **对于js文件而言，通常有四中引入js脚本的方式：**

2.  普通src引入js文件，这样的引入会造成阻塞，浏览器会立刻加载并执行指定的js脚本，在加载js的和执行js文件的时候，浏览器会挂起对html的渲染，所以会延迟页面的呈现速度，但是能够确定依赖关系。

3.  使用defer属性，这属性被设定用来通知浏览器该脚本将在文档完成解析后，触发 DOMContentLoaded 事件前执行。也能够确定依赖关系，在发现defer引入的js脚本后，会立即加载但是不执行，因此不会阻塞html进行解析，他的作用是在页面渲染完成后下载。

4.  使用async属性，这种方式前文有提到过，和defer一样，可以不阻塞html的解析。但是牺牲了依赖关系，到底是用defer还是async要看实际情况而定。

5.  动态加载js脚本，这种方式是现代化前端最受欢迎的，应用场景很多，比如SPA,对SPA而言，我们完全没必要一开始就加载好所有页面的js文件，只有在需要的时候import进来即可。

*TIP:当初始的 HTML 文档被完全加载和解析完成之后，DOMContentLoaded 事件被触发，而无需等待样式表、图像和子框架的完成加载。另一个不同的事件 *[load](https://developer.mozilla.org/en-US/docs/Mozilla_event_reference/load)* 应该仅用于检测一个完全加载的页面。
在使用 DOMContentLoaded 更加合适的情况下使用 *[load](https://developer.mozilla.org/en-US/docs/Mozilla_event_reference/load)* 是一个令人难以置信的流行的错误，所以要谨慎。注意：DOMContentLoaded 事件必须等待其所属script之前的样式表加载解析完成才会触发。*

![](/images/posts/2019-09-19-XNYH-loading-and-execution-of-css-and-js/e52b4f605e59e0e78a0cec8f585adf8b.png)

CSS阻塞
-------

1.  CSS head中阻塞页面的渲染

之前有提到过，在head标签中当CSS以link标签引入时，浏览器会等待后续的渲染，暂停去下载并解析对应的CSS。需要注意的是，因为在head标签内的CSS文件是加载并已经分析完成了之后才继续渲染的，所以后续的渲染就是带样式的。

所以推荐把css写在head中。

1.  CSS阻塞js的执行

在前面的CSS加载完成之前，后面的js执行是会被阻塞的。

1.  CSS不阻塞外部脚本的加载

加载和执行是有本质的区别的，所以要分清这一点和上一点的说法。CSS会阻塞的只是执行而不是加载。这一点也很好理解，因为后续的js可能会需要我们之前引入的样式信息，如果CSS还没有解析完成，js就不好得到CSS的样式信息了。

事实上，webkit浏览器有一个html-preloader-scaner这样一个类，或是说有一个预先扫描器，这个扫描器会预先扫描html文件的词语，对于需要下载的内容，浏览器就会并发的加载这些内容。

*TIP：可以这样理解，浏览器对同一个页面的解析是单进程的（但是浏览器是多进程的），他就和人一样，运算上不能同时处理多个
东西，但是我们下载东西（相当于网购的图书或资料）是不需要运算什么东西的，买完之后等就行了，我们在等新书邮到手之前，依旧可以看之前还没看过的书，等新书到了，我们就会迫不及待的打开新的图书进行阅读，那本老书就要等我们的新书看完了在回顾了。当然，这要假设我们在得到新书之后立即看新书，喜新厌旧的场景。*

*补充说明：写在head里的css和js（async和defer除外，他们的加载不会阻塞后面dom的渲染）无论是加载还是执行，都是会让浏览器渲染后续内容出现阻塞的，不会说head中css和js加载的很慢，在加载过程中先渲染dom然后等加载好之后在挂起执行。也就是说，head中的css和js没有下载并执行完毕，页面就永远都处于白屏状态。页面呈现的一瞬间，也必定是head中css和js下载和执行完之后的结果。*

Js阻塞
------

1.  直接引入的js阻塞页面的渲染

所谓直接引入的js，就是通过src引入的，没有使用defer和async特性。这种方式额很好理解，因为当前引入的js可能会存在操作dom的情况，如果我们一边操作dom，一边渲染后续的dom就很可能出现问题，所以浏览器必须要保证当前的dom情况已经确定不会在变更的时候再去执行js。

如果使用async和defer就不会阻塞后续页面的渲染的。

1.  js不阻塞资源的加载

js不阻塞资源的加载，但是阻塞后续页面的渲染，那可能就会产生一个问题：既然已经暂停了解析器对后续页面的渲染，那浏览器就不会知道之后还有那些资源需要加载，那不还是阻塞了资源的加载了么？其实并不是这样，还是之前所说的与扫描器的问题，浏览器在加载js脚本时会暂停js执行，用与扫描器去分析后面的资源有什么，如果有需要请求的静态资源就请求他，然后回过头来执行js代码，所以是不会阻塞后续资源的加载的。

1.  js顺序执行，阻塞后续的js逻辑执行

*TIP：补充说明，async确实是当js加载完之后就立即执行，即使前一个加载完成的js还没执行完，下一个加载好的js也会执行。而defer是当js加载完和DOM树加载完后立即执行，不过与async不同的是，当前一个加载完成的js还执行完毕之前，后一个js只能排队等待执行。*

*相同点是async和defer都不会阻塞页面渲染。所以如果需要等待dom,或者需要保证js的依赖关系，还不想让js的加载阻塞页面渲染，就可以使用defer。*

加载和执行的一些优化点
----------------------

1.  CSS样式表置顶

前面有说道，CSS放在head中，可以阻塞页面的渲染，因此能够防止页面加载出来出现样式闪动的问题。

1.  用link代替import

import是css内部的语法，是不会触发浏览器并发加载的机制的。另外import的加载是会在页面全部加载和渲染完成之后才会做响应的引入操作。所以相当于将css写在了dom的最低端，无疑会影响性能。所以应该避免使用import去引入css样式。

1.  js脚本置底

浏览器对于同于个host的同时请求数量是有限制的，在head中的css和js的加载和执行都会阻塞页面的渲染，如果js和css都在head中抢夺请求机会，无疑会拖慢页面呈现的速度，另外js脚本通常情况下不应该用于决定样式，所以在呈现页面的方面，css的重要性远远大于js，应该优先css的加载。

1.  合理使用js的异步加载能力

*TIPS：经过试验，现代浏览器对import方式引入的css和link无异。*
