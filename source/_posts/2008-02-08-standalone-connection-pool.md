---
layout: post
title: Standalone connection pool
date: 2008-02-08
comments: true
---

В одном из проектов на яве мне понадобился connection pool, без него
соединения с базой получались слишком частые.  После определенного
поиска обнаружил DBCP от apache и 
[статью на винграде](http://forum.vingrad.ru/sources/topic-157958.html)

DBCP мне не очень понравился, дело вкуса, конечно, но слишком много
зависимостей. Да и тяжеловат он для мелкого проекта. Vingrad-овский
connection pool - делает немного не то что хотелось кэширует
statements.. И отсутствует обработка случаев, когда, к примеру база
умрет (на время)..

Критика без конкретных предложений - критиканство. Так что взял и
написал как нравится самому. Вот что получилось:

<!-- more -->

## Итак. Общая идея

Программа использует connection pool по шаблону:

Инициализация:

{% codeblock lang:java %}
 ConnectionPool pool = ConnectionPool.getInstance();
 pool.setUsername("user1");<br /> pool.setPassword("bebebe");
 pool.setPath("jdbc:postgresql://hostname/main");
 pool.setClassName("org.postgresql.Driver");
 pool.setPoolsize(5); // держать 5 коннекций 
 if (! pool.init() ) System.exit(100);
{%endcodeblock%}

Тут все просто, берем экземпляр connectionpool-а (проект небольшой,
так что оформим его в виде *singleton*, устанавливаем параметры, и
если не удалось инициализировать, то не судьба и программу
выполнять...

И дальше, в том месте где используется соединение, что-то вроде:

{% codeblock lang:java %}

  Connection dbconnection = ConnectionPool.getConnection();
   try {
         ....   
   }
finaly
   {
       ConnectionPool.releaseConnection(dbconnection);       
   }
   
{%endcodeblock%}
   
   Тут, если посмотреть, то connection берется не из экземпляра
connection pool, а из статического метода в классе. И таким же образом
отдается.  Если у нас connection pool в виде singleton - зачем
переливать из пустого в порожнее, брать по очереди connection pool и
затем из него connection? Взять экземпляр connection pool я могу и сам
в статическом методе. Мелочь, но писать программы проще...

## Как это все реализoвано.


``` java
import java.sql.Connection;
import java.util.Queue;
import java.util.concurrent.ConcurrentLinkedQueue;
import java.util.logging.Logger;

public class ConnectionPool {
    private static ConnectionPool instance;
    
    private ConnectionPool() {
        connections =  new ConcurrentLinkedQueue<connection>();
    }
    
    public static ConnectionPool getInstance() {
        if (instance == null) instance = new ConnectionPool();
        return instance;
    }
    
    
    public static Connection getConnection() { 
        return getInstance().getInstanceConnection(); 
    }    
    
    public static void releaseConnection(Connection c) {
        getInstance().releaseInstanceConnection(c);    
    }    
    
/**
 * Выделить соединение из пула.
 * логика такая:
 * если не пустой - взять соединение из пула, проверить на валидность
 * и вернуть. невалидные соединения сразу убиваем.
 * если пул пустой - сделать новое соединение и вернуть его. Если не смогли
 * сделать новое соединение (типа DB сервер умер) - просто жалуемся в лог     
 * и возвращаем null
 */
    
    private Connection getInstanceConnection() {
        if (! alreadyInitialized){
            log.severe("improper using of db pool (getConnection without init)");
            return null;
        };
        Connection currentConnection;
        do {
            currentConnection = connections.poll();

            if (currentConnection == null) break; //пустой пул
              //  если дохлое соединение - не будем его использовать
            if (! validConnection(currentConnection)) {
                currentConnection = null;
            }
        } while (currentConnection == null);

        try{
            if (connections.size() == 0) {
                log.info("DB Pool depleted, add new connection");
                connections.add(java.sql.DriverManager.getConnection(path, userName,password));
                currentConnection = connections.poll();                
                if (! validConnection(currentConnection)) {
                    // не судьба. Сдаемся
                    throw new Exception("new connection also is not valid");
            }
           }
        } catch (Exception ie) {
            
            log.severe("cannot provide sql connection:"+ie);
            return null;
        };
        return currentConnection;
    }

    private void  releaseInstanceConnection(Connection c) {
        if (! alreadyInitialized){
            log.severe("improper using of db pool (releseConnection without init)");
            return;
        };
        if (getCurrentSize() >= getMaxPoolSize()) {
            try { c.close(); } catch (Exception e) {};
        } else if (! validConnection(c) ) {
            // Не будем возвращать в пул дохлое соединение
        } else {
            connections.offer(c);
        }
    }
    
    
    private int maxPoolSize = 5;

    public int
        getMaxPoolSize() {
        return maxPoolSize;
    }
    
    public void setPoolsize(int maxPoolSize) {
        if (!alreadyInitialized) {
            this.maxPoolSize = maxPoolSize;
        } else {
            log.severe("setPoolsize after init. ignoring");
        }
    }
    
    public int getCurrentSize() {
        return connections.size();
    }

    private String userName;
    private String password;
    private String path;
    private String className;

    public String getUserName() {
        return userName;
    }

    public void setUsername(String userName) {
        if (!alreadyInitialized) {
            this.userName = userName;
        } else {
            log.severe("setUserName after init. ignoring");
        }
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        if (!alreadyInitialized) {
            this.password = password;
        } else {
            log.severe("setPassword after init. ignoring");
        }
    }

    public String getPath() {
        return path;
    }

    public void setPath(String path) {
        if (!alreadyInitialized) {
            this.path = path;
        } else {
            log.severe("Path after init. ignoring");
        }
    }

    public String getClassName() {
        return className;
    }

    public void setClassName(String className) {
        if (!alreadyInitialized) {
            this.className = className;
        } else {
            log.severe("setClassName after init. ignoring");
        }
    }

    /**
     * init - Инициализация  пула сообщений
     * @return boolean - успешно ли инициализирован пул (true - все хорошо)
     */

    public synchronized boolean init() {
        log.entering("ConnectionPool","init");
        try {
            if (alreadyInitialized) {
                throw new  Exception("Double init of db pool");
            }

            Class.forName(this.className);
            for (int i=1; i<=5; i++){
                connections.add(java.sql.DriverManager
                                .getConnection(path, userName,password));
                  };
            alreadyInitialized = true;
            log.exiting("ConnectionPool","init");
            return true;
        } catch (Exception e) {
            log.severe("Cannot init db pool:"+e);
        }

        log.exiting("ConnectionPool","init");
        return false;
    }

    private static Logger log = Logger.getLogger("dbpool");

    /**
     *   Хранилище для коннекций
     */

    private ConcurrentLinkedQueue<connection> connections;

    
    /**
     * переменная предотвращающая двойную инициализацию DB pool
     */
    private boolean alreadyInitialized = false;
    /**
     * Проверка что соединение до сих пор живо. Если есть проблемы, мы его
     * просто убьем
     *
     *  Проверяем просто делая setAutoCommit. Если никаких exceptions
     *  не последовало, считаем что все в порядке.
     */

    private boolean validConnection(java.sql.Connection c) {
        try {
            boolean status = c.getAutoCommit();
            c.setAutoCommit(! status);
            c.setAutoCommit(status);
        } catch (Exception e)
            {
                try { c.close(); } catch (Exception notImportant) {};
                log.info("Dead sql connection detected");
                return false;
            }
        return true;
    }
}
```                  
                  
Несколько попутных замечаний: Держится 5 соединений (по умолчанию, может быть
изменено в setPoolsize() ), если программа просит *больше* соединений, будут
устанавливаться новые соединения с базой. Просто они будут закрываться, когда в
них пропадет нужда.  Так что poolsize - это сколько соединений будет повторно
использоваться.


