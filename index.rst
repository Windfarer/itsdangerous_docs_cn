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
<https://docs.djangoproject.com/en/dev/topics/signing/>`_。它也支持JSON Web Signatures (JWS)。
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

If you want to expire signatures you can use the :class:`TimestampSigner`
class which will additionally put in a timestamp information and sign it.
On unsigning you can validate that the timestamp did not expire:

>>> from itsdangerous import TimestampSigner
>>> s = TimestampSigner('secret-key')
>>> string = s.sign('foo')
>>> s.unsign(string, max_age=5)
Traceback (most recent call last):
  ...
itsdangerous.SignatureExpired: Signature age 15 > 5 seconds

Serialization
-------------

Because strings are hard to handle this module also provides a
serialization interface similar to json/pickle and others.  (Internally
it uses simplejson by default, however this can be changed by subclassing.)
The :class:`Serializer` class implements that:

>>> from itsdangerous import Serializer
>>> s = Serializer('secret-key')
>>> s.dumps([1, 2, 3, 4])
'[1, 2, 3, 4].r7R9RhGgDPvvWl3iNzLuIIfELmo'

And it can of course also load:

>>> s.loads('[1, 2, 3, 4].r7R9RhGgDPvvWl3iNzLuIIfELmo')
[1, 2, 3, 4]

If you want to have the timestamp attached you can use the
:class:`TimedSerializer`.

URL Safe Serialization
----------------------

Often it is helpful if you can pass these trusted strings to environments
where you only have a limited set of characters available.  Because of
this, itsdangerous also provides URL safe serializers:

>>> from itsdangerous import URLSafeSerializer
>>> s = URLSafeSerializer('secret-key')
>>> s.dumps([1, 2, 3, 4])
'WzEsMiwzLDRd.wSPHqC0gR7VUqivlSukJ0IeTDgo'
>>> s.loads('WzEsMiwzLDRd.wSPHqC0gR7VUqivlSukJ0IeTDgo')
[1, 2, 3, 4]

JSON Web Signatures
-------------------

Starting with “itsdangerous” 0.18 JSON Web Signatures are also supported.
They generally work very similar to the already existing URL safe
serializer but will emit headers according to the current draft (10) of
the JSON Web Signature (JWS) [``draft-ietf-jose-json-web-signature``].

>>> from itsdangerous import JSONWebSignatureSerializer
>>> s = JSONWebSignatureSerializer('secret-key')
>>> s.dumps({'x': 42})
'eyJhbGciOiJIUzI1NiJ9.eyJ4Ijo0Mn0.ZdTn1YyGz9Yx5B5wNpWRL221G1WpVE5fPCPKNuc6UAo'

When loading the value back the header will not be returned by default
like with the other serializers.  However it is possible to also ask for
the header by passing ``return_header=True``.
Custom header fields can be provided upon serialization:

>>> s.dumps(0, header_fields={'v': 1})
'eyJhbGciOiJIUzI1NiIsInYiOjF9.MA.wT-RZI9YU06R919VBdAfTLn82_iIQD70J_j-3F4z_aM'
>>> s.loads('eyJhbGciOiJIUzI1NiIsInYiOjF9.MA.wT-RZI9YU06R919VBdAf'
...         'TLn82_iIQD70J_j-3F4z_aM', return_header=True)
...
(0, {u'alg': u'HS256', u'v': 1})

“itsdangerous” only provides HMAC SHA derivatives and the none algorithm
at the moment and does not support the ECC based ones.  The algorithm in
the header is checked against the one of the serializer and on a mismatch
a :exc:`BadSignature` exception is raised.

.. _the-salt:

The Salt
--------

All classes also accept a salt argument.  The name might be misleading
because usually if you think of salts in cryptography you would expect the
salt to be something that is stored alongside the resulting signed string
as a way to prevent rainbow table lookups.  Such salts are usually public.

In “itsdangerous”, like in the original Django implementation, the salt
serves a different purpose.  You could describe it as namespacing.  It's
still not critical if you disclose it because without the secret key it
does not help an attacker.

Let's assume that you have two links you want to sign.  You have the
activation link on your system which can activate a user account and then
you have an upgrade link that can upgrade a user's account to a paid
account which you send out via email.  If in both cases all you sign is
the user ID a user could reuse the variable part in the URL from the
activation link to upgrade the account.  Now you could either put more
information in there which you sign (like the intention: upgrade or
activate), but you could also use different salts:

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

Only the serializer with the same salt can load the value:

>>> s2.loads(s2.dumps(42))
42

Responding to Failure
---------------------

Starting with itsdangerous 0.14 exceptions have helpful attributes which
allow you to inspect payload if the signature check failed.  This has to
be done with extra care because at that point you know that someone
tampered with your data but it might be useful for debugging purposes.

Example usage::

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
            # This payload is decoded but unsafe because someone
            # tampered with the signature.  The decode (load_payload)
            # step is explicit because it might be unsafe to unserialize
            # the payload (think pickle instead of json!)

If you don't want to inspect attributes to figure out what exactly went
wrong you can also use the unsafe loading::

    from itsdangerous import URLSafeSerializer
    s = URLSafeSerializer('secret-key')
    sig_okay, payload = s.loads_unsafe(data)

The first item in the returned tuple is a boolean that indicates if the
signature was correct.

Python 3 Notes
--------------

On Python 3 the interface that itsdangerous provides can be confusing at
first.  Depending on the internal serializer it wraps the return value of
the functions can alter between unicode strings or bytes objects.  The
internal signer is always byte based.

This is done to allow the module to operate on different serializers
independent of how they are implemented.  The module decides on the
type of the serializer by doing a test serialization of an empty object.


.. include:: ../CHANGES

API
---

Signers
~~~~~~~

.. autoclass:: Signer
   :members:

.. autoclass:: TimestampSigner
   :members:

Signing Algorithms
~~~~~~~~~~~~~~~~~~

.. autoclass:: NoneAlgorithm

.. autoclass:: HMACAlgorithm

Serializers
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

Exceptions
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
