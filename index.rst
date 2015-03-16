itsdangerous 中文文档
============

.. module:: itsdangerous

有时候你只是想向不被信任的环境发送一些数据，但是，如何安全的干这个事呢？
解决的方法就是签名。使用一个只有你自己知道的密钥，来加密签名你的数据，并把加密
后的数据传递给别人。当你取回数据时，你就可以确保没人篡改过这份数据。  

的确，接收者可以破译内容，来看看你的包裹里有什么，但是他们没法修改你的内容，
除非他们也有你的密钥。所以只要你保管好你的密钥，并且你的密钥足够复杂，
一切就OK了。

itsdangerous内部默认使用了HMAC和SHA1来签名，基于 `Django 签名模块
<https://docs.djangoproject.com/en/dev/topics/signing/>`_。它也支持JSON Web 签名 (JWS)。
这个库采用BSD协议，由Armin Ronacher编写，而大部分设计与实现的版权归Simon Willison和
其他的把这个库变为现实的Django爱好者们。

安装
------------

你可以从PyPI上直接安装这个库::

    pip install itsdangerous

适用案例
-----------------
-   在取消订阅某个通讯时，你可以在URL里序列化并且签名一个用户的ID。这种情况下
    你不需要生成一个一次性的token并把它们存到数据库中。在任何的激活账户
    的链接或类似的情形下，同样适用。
-   被签名的对象可以被存入cookie中或其他不被信任的来源，这意味着你不需要在服务端
    保存session，这将降低数据库读取的次数。
-   通常签名信息可以安全地往返与服务端与客户端之间，可以把这个特性用于将服务端的
    状态传递到客户端再传递回来。

签名接口
-----------------

最基本的接口是签名接口。 :class:`Signer` 类可以用来将一个签名附加到指定的字符串上：

>>> from itsdangerous import Signer
>>> s = Signer('secret-key')
>>> s.sign('my string')
'my string.wh6tMHxLgJqB6oY1uT73iMlyrOA'
    
签名会被加在字符串尾部，中间由句号 (``.``)分隔。验证字符串，使用 :meth:`~Signer.unsign`
方法：

>>> s.unsign('my string.wh6tMHxLgJqB6oY1uT73iMlyrOA')
'my string'

如果被签名的是一个unicode字符串，那么它将隐式地被转换成utf-8。
然而，在反签名时，你没法知道它原来是unicode还是字节串。

如果反签名失败了，将得到一个异常：

>>> s.unsign('my string.wh6tMHxLgJqB6oY1uT73iMlyrOX')
Traceback (most recent call last):
  ...
itsdangerous.BadSignature: Signature "wh6tMHxLgJqB6oY1uT73iMlyrOX" does not match

使用时间戳签名
--------------------------

如果你想要可以过期的签名，可以使用 :class:`TimestampSigner`类，它会加入时间戳信息并签名。
在反签名时，你可以验证时间戳有没有过期：

>>> from itsdangerous import TimestampSigner
>>> s = TimestampSigner('secret-key')
>>> string = s.sign('foo')
>>> s.unsign(string, max_age=5)
Traceback (most recent call last):
  ...
itsdangerous.SignatureExpired: Signature age 15 > 5 seconds

序列化
-------------

因为字符串难以被处理，本模块也提供了一个类似于json或pickle之类的序列化接口。
（它内部默认使用simplejson，但是可以由子类来改变。）
 :class:`Serializer`类实现了：

>>> from itsdangerous import Serializer
>>> s = Serializer('secret-key')
>>> s.dumps([1, 2, 3, 4])
'[1, 2, 3, 4].r7R9RhGgDPvvWl3iNzLuIIfELmo'

它当然也可以加载数据：

>>> s.loads('[1, 2, 3, 4].r7R9RhGgDPvvWl3iNzLuIIfELmo')
[1, 2, 3, 4]

如果你想要带一个时间戳，你可以用 :class:`TimedSerializer`类。

URL安全序列化
----------------------

如果能够向只有字符受限的环境中传递被信任的字符串的话，将会是很有用的。因此，
itsdangerous也提供了一个URL安全序列化工具：

>>> from itsdangerous import URLSafeSerializer
>>> s = URLSafeSerializer('secret-key')
>>> s.dumps([1, 2, 3, 4])
'WzEsMiwzLDRd.wSPHqC0gR7VUqivlSukJ0IeTDgo'
>>> s.loads('WzEsMiwzLDRd.wSPHqC0gR7VUqivlSukJ0IeTDgo')
[1, 2, 3, 4]

JSON Web 签名
-------------------

从“itsdangerous” 0.18版本开始，也支持了JSON Web签名。
They generally work very similar to the already existing URL safe
serializer but will emit headers according to the current draft (10) of
the JSON Web Signature (JWS) [``draft-ietf-jose-json-web-signature``].

>>> from itsdangerous import JSONWebSignatureSerializer
>>> s = JSONWebSignatureSerializer('secret-key')
>>> s.dumps({'x': 42})
'eyJhbGciOiJIUzI1NiJ9.eyJ4Ijo0Mn0.ZdTn1YyGz9Yx5B5wNpWRL221G1WpVE5fPCPKNuc6UAo'

在将值加载回来时，默认会像其他序列化器一样，header不会被返回。但是你可以通过传入
 ``return_header=True``参数来得到header。
Custom header fields can be provided upon serialization:

>>> s.dumps(0, header_fields={'v': 1})
'eyJhbGciOiJIUzI1NiIsInYiOjF9.MA.wT-RZI9YU06R919VBdAfTLn82_iIQD70J_j-3F4z_aM'
>>> s.loads('eyJhbGciOiJIUzI1NiIsInYiOjF9.MA.wT-RZI9YU06R919VBdAf'
...         'TLn82_iIQD70J_j-3F4z_aM', return_header=True)
...
(0, {u'alg': u'HS256', u'v': 1})

itsdangerous目前只提供HMAC SHA 的派生算法和不使用算法，不支持基于ECC的算法。
header中的算法将与序列化器中的进行核对，如果不匹配，将引发 :exc:`BadSignature`
异常。

.. _the-salt:

盐
--------

所有的类都接受一个盐的参数。这名字可能会误导你，因为通常你会认为，密码学中的盐
会是一个和被签名的字符串储存在一起的东西，用来防止彩虹表查找。这种盐是公开的。

在itsdangerous中，类似Django中的原始实现，盐是为了一个截然不同的目的而产生的。
你可以将它解释成命名空间。如果你泄露了它，也不是很严重的问题，因为没有密钥的话，
它对攻击者没什么帮助。

让我们假设你想签名两个链接。你的系统有个激活链接，用来激活一个用户账户，
然后你有一个升级链接，可以让一个用户账户升级为付费用户，这两个链接使用email发送。
在这两种情况下，如果你签名的都是用户ID，那么该用户可以在激活账户和升级账户时，
复用URL的可变部分。现在你可以在你签名的地方加上更多信息（如升级或激活的意图），
但是你也可以用不同的盐：

>>> s1 = URLSafeSerializer('secret-key', salt='activate-salt')
>>> s1.dumps(42)
'NDI.kubVFOOugP5PAIfEqLJbXQbfTxs'
>>> s2 = URLSafeSerializer('secret-key', salt='upgrade-salt')
>>> s2.dumps(42)
'NDI.7lx-N1P-z2veJ7nT1_2bnTkjGTE'
>>> s2.loads(s1.dumps(42))
Traceback (most recent call last):
  ...
itsdangerous.BadSignature: Signature "kubVFOOugP5PAIfEqLJbXQbfTxs" does not match

只有使用相同盐的序列化器才能成功把值加载出来：

>>> s2.loads(s2.dumps(42))
42

对失败的响应
---------------------

从itsdangerous 0.14版本开始，异常会有一些有用的属性，可以允许你在签名检查失败时，
检查你的数据。这里必须极其小心，因为这个时候，你知道有某人修改了你的数据。
但这可能对你debug很有帮助。

示例用法::

    from itsdangerous import URLSafeSerializer, BadSignature, BadData
    s = URLSafeSerializer('secret-key')
    decoded_payload = None
    try:
        decoded_payload = s.loads(data)
        # This payload is decoded and safe
    except BadSignature, e:
        encoded_payload = e.payload
        if encoded_payload is not None:
            try:
                decoded_payload = s.load_payload(encoded_payload)
            except BadData:
                pass
            # 这里的数据被解码出来，但是不是安全的，因为有某人改动了签名。
            # 解码步骤(load_payload)是显式的，因为将数据反序列化可能是不安全的
            #（请设想被解码的不是json,而是pickle）

如果你不想检查属性来指出到底是哪里出错了，你也可以使用不安全的加载方式::

    from itsdangerous import URLSafeSerializer
    s = URLSafeSerializer('secret-key')
    sig_okay, payload = s.loads_unsafe(data)

返回的元组中第一项是一个布尔值，表明了签名是否是正确的。

Python 3 提示
--------------
在Python 3中，itsdangerous的接口在一开始可能让人困扰。基于它包裹的内部的序列化器，
函数返回值不一定是unicode字符串还是字节对象。内置的签名器总是基于字节的。

这是为了允许模块操作不同的序列化器，独立于它们是如何被实现的。模块通过执行一个
空对象的序列化，来决定使用哪种序列化器。

.. include:: ../CHANGES

API
---

签名器
~~~~~~~

.. autoclass:: Signer
   :members:

.. autoclass:: TimestampSigner
   :members:

签名算法
~~~~~~~~~~~~~~~~~~

.. autoclass:: NoneAlgorithm

.. autoclass:: HMACAlgorithm

序列化器
~~~~~~~~~~~

.. autoclass:: Serializer
   :members:

.. autoclass:: TimedSerializer
   :members:

.. autoclass:: JSONWebSignatureSerializer
   :members:

.. autoclass:: TimedJSONWebSignatureSerializer
   :members:

.. autoclass:: URLSafeSerializer

.. autoclass:: URLSafeTimedSerializer

异常
~~~~~~~~~~

.. autoexception:: BadData
   :members:

.. autoexception:: BadSignature
   :members:

.. autoexception:: BadTimeSignature
   :members:

.. autoexception:: SignatureExpired
   :members:

.. autoexception:: BadHeader
   :members:

.. autoexception:: BadPayload
   :members:

Useful Helpers
~~~~~~~~~~~~~~

.. autofunction:: base64_encode

.. autofunction:: base64_decode
