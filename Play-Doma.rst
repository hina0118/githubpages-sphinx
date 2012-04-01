=========================
Play-Doma Module
=========================

----

概要
====

Play framework 1.x用Domaモジュール。

使用しているライブラリについては以下のリンクを参照してください。

* `Play framework 1.2.4 <http://www.playframework.org/>`_
* `Doma 1.21.1 <http://doma.seasar.org/>`_

----

モジュールの設定方法
====================

１．GitHubからモジュールをcloneします。

   git clone git://github.com/hina0118/play-doma.git

２．cloneしたモジュールを **dependencies.yml** に追記します。

.. code-block:: yaml
   
   require:
       - play
       - doma -> doma
   
   repositories:
       - My modules:
           type:       local
           artifact:   ${application.path}/../../../[module]
           contains:
               - doma

３． **play dependencies** または **play deps** を実行するとモジュールが追加されます。

----

Eclipseプロジェクトの生成
=========================

**play eclipsify** 、 **play ec** にすぐにDomaを使用するための設定を行う処理を追加しています。

Play-Domaモジュールを取り込んだ状態でEclipseプロジェクトを生成すると、Eclipseに取り込んですぐにDomaを使用することができます。

以下の設定が追加された状態でEclipseにプロジェクトを取り込むことができます。

* aptを使用可能な状態にする
* アプリケーションのlibディレクトリ、モジュールのlibディレクトリの順に探索して先に見つかったDomaのjarをfactorypathentryに設定
* .apt_generatedディレクトリをaptが出力したソースコードの出力先に指定

----

Doma設定クラス
==============

Doma設定クラスは以下のように作成します。

Play frameworkと同じLoggerにログを出力するために **play.modules.doma.PlayLogger** を用意していますので、必要に応じて切り替えてください。

.. code-block:: java

   import javax.sql.DataSource;
   
   import org.seasar.doma.jdbc.DomaAbstractConfig;
   import org.seasar.doma.jdbc.JdbcLogger;
   import org.seasar.doma.jdbc.dialect.Dialect;
   import org.seasar.doma.jdbc.dialect.H2Dialect;
   import org.seasar.doma.jdbc.tx.LocalTransaction;
   import org.seasar.doma.jdbc.tx.LocalTransactionalDataSource;
   
   import play.db.DB;
   
   import play.modules.doma.PlayLogger;
   
   public class AppConfig extends DomaAbstractConfig {
   
       protected static final LocalTransactionalDataSource dataSource = createDataSource();
   
       protected static final Dialect dialect = new H2Dialect();
   
       protected static final JdbcLogger jdbcLogger = new PlayLogger();
   
       @Override
       public DataSource getDataSource() {
           return dataSource;
       }
   
       @Override
       public Dialect getDialect() {
           return dialect;
       }
   
       @Override
       public JdbcLogger getJdbcLogger() {
           return jdbcLogger;
       }
   
       protected static LocalTransactionalDataSource createDataSource() {
           return new LocalTransactionalDataSource(DB.datasource);
       }
   
       public static LocalTransaction getLocalTransaction() {
           return dataSource.getLocalTransaction(jdbcLogger);
       }
   }

----

Controllerクラス
================

Daoクラスの生成
---------------

Daoクラスはnew演算子で実装クラスをインスタンス化することもできますが、以下のように **javax.inject.Inject** アノテーションを注釈するとモジュール側で実装クラスを探しだしてInjectします。

.. code-block:: java

   @javax.inject.Inject
   private static UserDao userDao;

.. note ::

   Controllerクラスに定義したstaticフィールドに対して有効です。


トランザクション管理
--------------------

リクエストの開始、終了、エラー発生時にモジュール側でローカルトランザクションの開始、コミット、ロールバックを制御しています。

そのため、Controllerクラスでは以下のようにシンプルに記述することができます。

.. code-block:: java

   public static void index() {
       List<User> users = userDao.select();
       render(users);
   }

リクエストパラメータのバインド
------------------------------

以下のHTMLのようにuser.emailと名前を付けられた部品の入力値をDomaのEntity、Domainに対して直接バインドすることができます。

.. code-block:: html

   #{form @Application.create()}
   <label>MAIL:</label><input type="text" name="user.email">
   <label>PASS:</label><input type="password" name="user.password">
   <label>NAME:</label><input type="text" name="user.fullname">
   <input type="submit" value="insert">
   #{/form}

Userクラスが以下のように定義されている場合、

.. code-block:: java

   @org.seasar.doma.Entity
   public class User {
       Integer id;         // IDは自動生成
       Email email;        // Domainクラス
       Password password;  // Domainクラス
       String fullname;

       // setter and getter
   }

Userクラスには上記HTMLで入力された値が既にバインドされてるのでそのまま使用できます。

.. code-block:: java

   public static void create(User user) {
       userDao.insert(user);
       index();
   }

----

.apt_generatedについて
======================

.apt_generatedディレクトリに出力されるソースコードはアプリケーション実行時に参照されるようになっていますのでリリース時などは他のソースコードと同じように扱う必要があります。

