---
layout: post
title: Performance degradation for gcc
date: 2008-02-29
comments: true 
---

Просмативая свои старые архивы, нашел мелкую программу sieve - решето
Эратосфена, которая была написана на разных языках. И там же валялся бинарник
этой программы, скомпилированной gcc в далеком (по моему 2002) году. Запустил
из интереса - получилось

<!-- more -->

> 103763 iterations in 10.000000 seconds. 

Скомпилировал текущей версией gcc из ubuntu - получилось 

> 96992 iterations in 10.000000 seconds
    
Вот тебе бабушка и Юрьев день! Какой был в 2002 gcc не помню, file a.out покзывает

> a.out: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), for GNU/Linux 2.0.0, 
>  dynamically linked (uses shared libs), not stripped

  
Теперь у нас gcc version 4.1.3 20070929 (prerelease) (Ubuntu 4.1.2-16ubuntu2), file
показывает

> a.out: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), for GNU/Linux 2.6.8, 
>  dynamically linked (uses shared libs), not stripped
    
То есть все честно.. Но медленнее..


