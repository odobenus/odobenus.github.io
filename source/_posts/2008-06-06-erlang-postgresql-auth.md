---
layout: post
title: erlang & postgresql auth
date: 2008-06-06
comments: true
---


Для прямого доступа из эрланга в postgresql (минуя odbc) есть хорошая библиотечка, написанная  Martin Carlsson из Erlang Training & Consulting ltd 
[Здесь]( http://erlang-consulting.com/aboutus/opensource.html). 

У нее есть один недостаток - она рассчитана только на md5  authentication в постгресе.

Если приложение разрабатывается с нуля, то md5 auth - не представляет проблемы, как захотим, так и настроим, но в нашем случае уже есть 
большой и толстый postgresql сервер, со своими админами и правилами выделения учетных записей. В этом сервере метод доступа у большинства аккаунтов - 
password. И никак это не поправишь. В отличие от технических проблем - административные проблемы плохо поддаются решению..

Пришлось прочитать хорошенько документацию postgresql, раздел где описан протокол, и поправить erlang-овскую библиотеку. 
Вот, чтобы не забыть, что я там наисправлял, и  
выкладываю diff

```diff
diff -Naur oldsrc/psql_logic.erl src/psql_logic.erl
--- oldsrc/psql_logic.erl       2006-09-18 17:46:20.000000000 +0200
+++ src/psql_logic.erl  2008-03-12 17:41:48.000000000 +0100
@@ -139,6 +139,12 @@
     AuthDigest = psql_protocol:md5digest(State#state.digest, Salt),
     psql_connection:command(State#state.connection, {send, AuthDigest}),
     {next_state, authentication, State};
+
+authentication({psql, authentication, <<0,0,0,3>>}, State) ->
+    CTPassw = psql_protocol:ctpasswd(State#state.password),
+    psql_connection:command(State#state.connection, {send, CTPassw}),
+    {next_state, authentication, State};
+
authentication({psql, authentication, <<0,0,0,0>>}, State) ->
     {next_state, setup, State};
authentication({psql, error, Error}, State) ->
```

```diff
--- oldsrc/psql_protocol.erl    2006-09-18 17:46:24.000000000 +0200
+++ src/psql_protocol.erl       2008-03-12 17:41:31.000000000 +0100
@@ -21,7 +21,7 @@
         to_string/1]).

 -export([authenticate/4,
-        md5digest/2,
+        md5digest/2,ctpasswd/1,
         copy_done/0,
         q/1]).

@@ -65,6 +65,9 @@
     Auth = md5([Digest, Salt]),
     {password_message, <<"md5", Auth/binary, 0:8>>}.

+ctpasswd(Passw) ->
+    PB = list_to_binary(Passw),
+    {password_message, << PB/binary,0:8 >>}.
+
copy_done() ->
     {'copy_done', []}.
```
