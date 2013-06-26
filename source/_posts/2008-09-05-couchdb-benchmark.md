---
layout: post
title: couchdb benchmark еще раз
date: 2008-09-05
comments: true
---

Попробоавл тот же тест, но 10 процессов по 10000 записей. Суммарные цифры -
100_000 записей за 2308 сек. То есть 43 записи/сек. Примерно так же как и в
прошлом тесте. Похоже, быстрее уде не разогнать.

Размер базы при 100000 записях - 0.6G

<!-- more -->
##Comments

*Marat*> angry-elf, не поделитесь своими результатами? У меня с Вашим тестом получилось 473 вставки в сек.

*angry-elf*> А еще можно не через create их делать (я так понимаю, документ
создаётся, сохраняется и тут же возвращается, что дико не оптимально), а через
update, да еще и не по одному за раз... То скорость растёт на порядки.
Судя по тому, что я вижу в iostat, что единичные документы, что блок,
порождают примерно одинаковый io, что есть странно. Т.е. 1 процесс, пишущий 1
документ за раз - около 2мб/с нагрузки. 

2 процесса, соответственно, 4 мб/с. А если писать сразу по 100, то тоже.. 2мб/с при выросшей почти в 100 раз
скорости. 

``` python
from couchdb import Server,Document
import time
server = Server('http://localhost:5984/')
try:
   server.create('benchmark')
except:
   pass
db = server['benchmark']
benchstart = time.time()
print "Begin test %s"%benchstart

documents = []
chunk_size = 100

for i in range(1, 100000):
     documents.append(Document(type = 'BechRecord', intfld = i, charfld = ('SomeName%s' % i)))
     if len(documents) == chunk_size:
        db.update(documents)
        documents = []
     if i % (chunk_size * 2) == 0:
          bench_offset = time.time() - benchstart
          print "%d\t%d\t%.2f/s " % (bench_offset, i, 1.0 * i / bench_offset)
print "End test, %d sec, %.2f/s" % (time.time() - benchstart, 1.0 * i / (time.time() - benchstart))
```

P.S. Надеюсь, понятно будет. Не осилил найти pre/code-тэги...

