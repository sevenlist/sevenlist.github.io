---
layout: post
title: "A Groovy Java Puzzler: Setting a Property When Its Setter Is Overloaded"
date: 2014-08-24 12:40:00 +0200
comments: true
categories:
---

Trying out [Spring's email support](http://docs.spring.io/spring/docs/4.0.6.RELEASE/spring-framework-reference/html/mail.html), I wanted to
send a simple message to a single recipient using code like:
``` groovy
SimpleMailMessage mailMessage = new SimpleMailMessage()
mailMessage.to = 'me@trash-mail.com'
// ...
mailSender.send(mailMessage)
```
To my astonishment this threw an exception:
``` groovy
javax.mail.internet.AddressException: Missing local name in string ``@''
  at javax.mail.internet.InternetAddress.checkAddress(InternetAddress.java:1209)
	at javax.mail.internet.InternetAddress.parse(InternetAddress.java:1091)
	at javax.mail.internet.InternetAddress.parse(InternetAddress.java:633)
	at javax.mail.internet.InternetAddress.parse(InternetAddress.java:610)
	at org.springframework.mail.javamail.MimeMessageHelper.parseAddress(MimeMessageHelper.java:707)
	at org.springframework.mail.javamail.MimeMessageHelper.setTo(MimeMessageHelper.java:593)
	at org.springframework.mail.javamail.MimeMailMessage.setTo(MimeMailMessage.java:109)
	at org.springframework.mail.SimpleMailMessage.copyTo(SimpleMailMessage.java:194)
	at org.springframework.mail.javamail.JavaMailSenderImpl.send(JavaMailSenderImpl.java:305)
	at org.springframework.mail.javamail.JavaMailSenderImpl.send(JavaMailSenderImpl.java:297)
	at org.springframework.mail.MailSender$send.call(Unknown Source)
	...
```
Taking a closer look, I recognized that the `setTo` method of `SimpleMailMessage` is overloaded as follows:
``` groovy
public void setTo(String to)
public void setTo(String[] to)
```
<!--more-->
So the second method allows for passing a list of recipients.

My intuition told me to check the bytecode the Groovy compiler has generated for setting the `to` property, using the wonderful [Procyon](https://bitbucket.org/mstrobel/procyon/wiki/Java%20Decompiler) decompiler:
```groovy
ScriptBytecodeAdapter.setProperty((Object)"me@trash-mail.com", (Class)null, (Object)mailMessage, "to");
```
Not having the book at hand at that time but vaguely remembering a [Java Puzzler](http://www.amazon.com/Java%C2%BF-Puzzlers-Traps-Pitfalls-Corner/dp/032133678X/ref=sr_1_1?ie=UTF8&qid=1408215865&sr=8-1&keywords=java+puzzler)
for passing `null` to an overloaded method, I searched the _Java Language Specification_ for how methods are chosen in my case.
I found out and refreshed my memory that Java [chooses the most specific method](http://docs.oracle.com/javase/specs/jls/se7/html/jls-15.html#jls-15.12.2.5).

As an array parameter is more specific than its single reference counterpart, the `setTo(String[])` method is called.

With the Groovy logic to transform a String to a List of that String's characters when that String is passed to a method
with a String array parameter, this results figuratively in an invocation like:
``` groovy
mailMessage.setTo(['m', 'e', '@', 't', 'r', 'a', 's', 'h', '-', 'm', 'a', 'i', 'l', '.', 'c', 'o', 'm'])
```
As each of those array entries is handled like a recipient, each is passed within a loop to InternetAddress.checkAddress(..).
The looping terminates while its third iteration as the "recipient" `@` is passed to checkAddress(..) - the AddressException is thrown, stating
`Missing local name in string ``@''`. Finally, this message makes more sense, doesn't it?

By the way, to solve the problem I just wrote:
``` groovy
mailMessage.to = ['me@trash-mail.com']
```

All in all, isn't that a groovy Java Puzzler?
