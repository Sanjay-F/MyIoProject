title: 源码探索系列4---数据库ORM框架之Ormlite解析
date: 2015-12-16 02:37
tags: [android,源码,OrmLite,ORM]
categories: android
------------------------------------------

在做项目时候，有使用Ormlite来做数据库部分，以提高开发速度。

>  OrmLite - Lightweight Object Relational Mapping (ORM) Java Package

这个ORM框架在一般数据量不大的情况下，是一个很好的工具，他封装得很好，使用起来也很方便！
相比于原始的`Sqlite`来说，使用后，很容易回不去了，
因为你不再需要敲一堆的sql语句，担心那个字段写错，处理cursor，写各种查询等等！
重要的是在增加一些字段属性时候，不再需要改到哭了！
<!--more-->

他提供了很多`Builder模式`设计的接口，方便我们进行`CRUD操作`，这真的用起来很友好啊。
当然这写ORM都是这样的友好

如果你用的还是`原生的Sqlite`，最数据的速度没有那么高的要求（虽然我们实际也经常是开一个线程去加载的），没听说过ORM的，建议你采用下`Ormlite`。
它类似于Hibernate和Mybatis等等，在提高开发效率的同时，也可以有效避免数据库操作对应用带来的潜在影响。不要再去写那些 Magic Number和Xxx了 ，我对他的广告就到这里吧，对于他的使用，可以查看这篇文章


现在刚好有点时间，就开始解析整个项目的代码，看下他背后是怎么做到这些功能的！
 
# 起航 -- 创建表
Ormlite版本：V4.48

万事开头难，这个库已经挺成熟也庞大的了。就让我们从创建表格开始说起吧
如果你用过，对这句应该很熟悉

	@Override
	    public void onCreate(SQLiteDatabase db, ConnectionSource connectionSource) {
	        try {
	            TableUtils.createTable(connectionSource, User.class);
	}
那他背后到底怎么做到创建表的呢？
我猜可能就是根据获取类的注解，来获取各个字段，最后拼凑成sql语句的，让我看下源码吧


	public static <T> int createTable(ConnectionSource connectionSource, Class<T> dataClass) throws SQLException {
        return createTable(connectionSource, dataClass, false);
    }
	
	private static <T, ID> int createTable(ConnectionSource connectionSource, Class<T> dataClass, boolean ifNotExists) throws SQLException {
        Dao dao = DaoManager.createDao(connectionSource, dataClass);
        if(dao instanceof BaseDaoImpl) {
            return doCreateTable(connectionSource, ((BaseDaoImpl)dao).getTableInfo(), ifNotExists);
        } else {
            TableInfo tableInfo = new TableInfo(connectionSource, (BaseDaoImpl)null, dataClass);
            return doCreateTable(connectionSource, tableInfo, ifNotExists);
        }
    }
他会先去创建dao，同时这个dao是用`容器单例`的模式来做的，所以后面我们继续调用这个Dao的时候，可以快速的获得，得到Dao后，他去创建`Table`，相关代码如下

	 public static synchronized <D extends Dao<T, ?>, T> D createDao(
					 ConnectionSource connectionSource, Class<T> clazz) throws SQLException {
	 
      
            DaoManager.ClassConnectionSource key = new DaoManager.ClassConnectionSource(connectionSource, clazz);
            
            Dao dao = lookupDao(key);
            if(dao != null) {
                return dao;
            }  
            ...
              
	}              

	private static <T> Dao<?, ?> lookupDao(DaoManager.ClassConnectionSource key) {
        if(classMap == null) {
            classMap = new HashMap();
        }

        Dao dao = (Dao)classMap.get(key);
        return dao == null?null:dao;
    }

创建表：

	private static <T, ID> int doCreateTable(ConnectionSource connectionSource, TableInfo<T, ID> tableInfo, boolean ifNotExists) throws SQLException {
        DatabaseType databaseType = connectionSource.getDatabaseType();
        logger.info("creating table \'{}\'", tableInfo.getTableName());
        ArrayList statements = new ArrayList();
        ArrayList queriesAfter = new ArrayList();
        addCreateTableStatements(databaseType, tableInfo, statements, queriesAfter, ifNotExists);
        DatabaseConnection connection = connectionSource.getReadWriteConnection();

        int var8;
        try {
            int stmtC = doStatements(connection, "create", statements, false, databaseType.isCreateTableReturnsNegative(), databaseType.isCreateTableReturnsZero());
            stmtC += doCreateTestQueries(connection, databaseType, queriesAfter);
            var8 = stmtC;
        } finally {
            connectionSource.releaseConnection(connection);
        }

        return var8;
    }
这里有需要注意的

 1. **addCreateTableStatements(databaseType, tableInfo, statements, queriesAfter, ifNotExists);**
 这句负责去生成我们要执行的sql语句。
 
		 private static <T, ID> void addCreateTableStatements(DatabaseType databaseType, TableInfo<T, ID> tableInfo, List<String> statements, List<String> queriesAfter, boolean ifNotExists) throws SQLException {
	        StringBuilder sb = new StringBuilder(256);
	        sb.append("CREATE TABLE ");
	        if(ifNotExists && databaseType.isCreateIfNotExistsSupported()) {
	            sb.append("IF NOT EXISTS ");
	        }
	
	        databaseType.appendEscapedEntityName(sb, tableInfo.getTableName());
	        sb.append(" (");
	        ArrayList additionalArgs = new ArrayList();
	        ArrayList statementsBefore = new ArrayList();
	        ArrayList statementsAfter = new ArrayList();
	        boolean first = true;
	        FieldType[] i$ = tableInfo.getFieldTypes();
	        int arg = i$.length;
	
	        for(int i$1 = 0; i$1 < arg; ++i$1) {
	            FieldType fieldType = i$[i$1];
	            if(!fieldType.isForeignCollection()) {
	                if(first) {
	                    first = false;
	                } else {
	                    sb.append(", ");
	                }
	
	                String columnDefinition = fieldType.getColumnDefinition();
	                if(columnDefinition == null) {
	                    databaseType.appendColumnArg(tableInfo.getTableName(), sb, fieldType, additionalArgs, statementsBefore, statementsAfter, queriesAfter);
	                } else {
	                    databaseType.appendEscapedEntityName(sb, fieldType.getColumnName());
	                    sb.append(' ').append(columnDefinition).append(' ');
	                }
	            }
	        }
	
	        databaseType.addPrimaryKeySql(tableInfo.getFieldTypes(), additionalArgs, statementsBefore, statementsAfter, queriesAfter);
	        databaseType.addUniqueComboSql(tableInfo.getFieldTypes(), additionalArgs, statementsBefore, statementsAfter, queriesAfter);
	        Iterator var16 = additionalArgs.iterator();
	
	        while(var16.hasNext()) {
	            String var15 = (String)var16.next();
	            sb.append(", ").append(var15);
	        }
	
	        sb.append(") ");
	        databaseType.appendCreateTableSuffix(sb);
	        statements.addAll(statementsBefore);
	        statements.add(sb.toString());
	        statements.addAll(statementsAfter);
	        addCreateIndexStatements(databaseType, tableInfo, statements, ifNotExists, false);
	        addCreateIndexStatements(databaseType, tableInfo, statements, ifNotExists, true);
	    }
 2. **int stmtC = doStatements(connection, "create", statements, false, **
 **databaseType.isCreateTableReturnsNegative(), databaseType.isCreateTableReturnsZero());**
 这句负责去执行我们的sql语句 
 
		 private static int doStatements(DatabaseConnection connection, String label, Collection<String> statements, boolean ignoreErrors, boolean returnsNegative, boolean expectingZero) throws SQLException {
		 
	        int stmtC = 0;
	
	        for(Iterator i$ = statements.iterator(); i$.hasNext(); ++stmtC) {
	            String statement = (String)i$.next();
	            int rowC = 0;
	            CompiledStatement compiledStmt = null;
	 
	                compiledStmt = connection.compileStatement(statement, StatementType.EXECUTE, noFieldTypes, -1);
	                rowC = compiledStmt.runExecute();
	                logger.info("executed {} table statement changed {} rows: {}", label, Integer.valueOf(rowC), statement); 
	            }
	
	           ...
	
	        return stmtC;
	    }
	    
	其中 `connection.compileStatement(）`，这个`connection`从前面可以看到在
 `DatabaseConnection connection =connectionSource.getReadWriteConnection();` 
 是`ConnectionSource`给他的。而 `DatabaseConnection`是一个接口，他的具体实现是`AndroidConnectionSource`，出现在`OrmLiteSqliteOpenHelper` 的构造函数里面。
	
	   我们继续看下这个getReadWriteConnection() 里面返回了什么，因为这是把`Ormlite`和`Sqlite`连接上的地方
	   
	    public DatabaseConnection getReadWriteConnection() throws SQLException {
	    
                    ...
                    
              SQLiteDatabase db;
              if(this.sqliteDatabase == null) { 
                       ....
                  db = this.helper.getWritableDatabase();
              }else{	                     
                  db = this.sqliteDatabase;
              }

              this.connection = new AndroidDatabaseConnection(db, true, this.cancelQueriesEnabled);
              if(connectionProxyFactory != null) {
                  this.connection = connectionProxyFactory.createProxy(this.connection);
              } 
              
	          return this.connection;
	       
	    }
我们清楚的看到，他把我们的`SQLiteDatabase`给抓了，重新包装成了`AndroidDatabaseConnection`类，然后返回，而这个是实现了`DatabaseConnection`接口的，感觉像一个代理模型啊。
既然找到别后实际干活的人，那我们继续看下我们原来的那个
`compiledStmt = connection.compileStatement(statement, StatementType.EXECUTE, noFieldTypes, -1);`
里面到底做了什么。

		public CompiledStatement compileStatement(String statement, StatementType type, FieldType[] argFieldTypes, int resultFlags) {
		
		        AndroidCompiledStatement stmt = new AndroidCompiledStatement(statement, this.db, type, this.cancelQueriesEnabled);
		 
		        return stmt;
		    }

  我们看到的是他只是简单的把他封装成了一个`AndroidCompiledStatement`类，同时把我们的`SQLiteDatabase  DB`也送给他，交给他去干活 。另外他的命名也很有规则啊，涉及到具体于安卓相关的干活的人，都加个AndroidXXX的格式，这个则具体实现了CompiledStatement接口。
  
		  public class AndroidCompiledStatement implements CompiledStatement 
我们继续看那个`runExecute()`方法的内容

		public int runExecute() throws SQLException {
           ...
	       return execSql(this.db, "runExecute", this.sql, this.getArgArray()); 
	    }
 好啦，看了这么久，终于看到一个背后具体干活去执行语句，创建我们的表的地方啦
 
	    static int execSql(SQLiteDatabase db, String label, String finalSql, Object[] argArray) {		 
		  
		    db.execSQL(finalSql, argArray);
		   ...
		
	        SQLiteStatement stmt = null;		
	        int result;
	     
            stmt = db.compileStatement("SELECT CHANGES()");
            result = (int)stmt.simpleQueryForLong();	      
            if(stmt != null) {
                stmt.close();
            }		
        
	        return result;
	    }
到这里，我们的第一步，创建表的过程就了解。
他把所有的关于操作数据库的部分用接口写处理啊，然后加一个AndroidXXX，来表示具体实现，由它负责去干活，从而解耦，很好的面向接口编程的方式。

#四大神兽---CRUD

看完了创建表的功能，接下来我们了解下基本的四大神兽增删查改---CRUD 
## 增加(Create)
先从我们的看到的第一句开始做入口

    DataHelper helper = new DataHelper(this);        
    helper.getUserDao().create(mUser);
        
这里，我们的Dao类也是一个借口，我们重新找真正干活的，不知道你在上面的流程有没注意到他的实现类是出现过的，就在我们创建表的时候，看到了一个`BaseDaoImpl`，居然不是AndroidDao的样子，不科学.

	private static <T, ID> int createTable(ConnectionSource connectionSource, Class<T> dataClass, boolean ifNotExists) throws SQLException {
        Dao dao = DaoManager.createDao(connectionSource, dataClass);
        if(dao instanceof BaseDaoImpl) {
            return doCreateTable(connectionSource, ((BaseDaoImpl)dao).getTableInfo(), ifNotExists);
        } else {
            TableInfo tableInfo = new TableInfo(connectionSource, (BaseDaoImpl)null, dataClass);
            return doCreateTable(connectionSource, tableInfo, ifNotExists);
        }
    }
就是这里提示了我们是BaseDaoImpl，我们就去这里，看下背后干了什么

    public int create(T data) throws SQLException {
    
        ...
         
        DatabaseConnection connection1 = this.connectionSource.getReadWriteConnection();

        int var3;
        try {
            var3 = this.statementExecutor.create(connection1, data, this.objectCache);
        } finally {
            this.connectionSource.releaseConnection(connection1);
        }

        return var3;  
    }
我们看到了些熟悉的东西，`DatabaseConnection connection1 = this.connectionSource.getReadWriteConnection();`
获得与数据库的连接，然后`this.statementExecutor.create(connection1, data, this.objectCache);`这个执行语句。

	public int create(DatabaseConnection databaseConnection, T data, ObjectCache objectCache) throws SQLException {
        if(this.mappedInsert == null) {
            this.mappedInsert = MappedCreate.build(this.databaseType, this.tableInfo);
        }

        return this.mappedInsert.insert(this.databaseType, databaseConnection, data, objectCache);
    }
  这里有一点需要注意的，就是这个`MappedCreate.build()`，他负责的工作和他名字一样，是做Map工作的，`build（）`函数他根据传给他的表信息`tableInfo`，生成了相关的插入Sql语句。交给后面去执行
    经过处理后，我们继续看插入操作
	
    public int insert(DatabaseType databaseType, DatabaseConnection databaseConnection, T data, ObjectCache objectCache) throws SQLException {
         
        ...
            
       rowC = databaseConnection.insert(this.statement, var14, this.argFieldTypes, keyHolder);
        ...             
 
    }
   很好，这个`databaseConnection`我们知道，前面遇到过他了，`AndroidDatabaseConnection`是他的具体实现，我们去他的insert看看，很可能就是实际插入的地方
	
	public int insert(String statement, Object[] args, FieldType[] argFieldTypes, GeneratedKeyHolder keyHolder) throws SQLException {
	
        SQLiteStatement stmt = null;
        byte var9; 
        
        stmt = this.db.compileStatement(statement);
        this.bindArgs(stmt, args, argFieldTypes);
        long e = stmt.executeInsert();
 
        ...
        byte result = 1;      
        var9 = result;     
         ...
        return var9;
    }
    
看到这里，整个语句也就结束了， 他这里使用的方式是
`stmt = this.db.compileStatement(statement);`然后加参数bindArgs(）的形式去执的。

## 读取(Retrieve)

下次补充。。。
这个和前面不一样

## 更新(Update)

前面的步骤和Create的差不多，就不多余，在我们的最后的AndroidDatabaseConnection里面有一点不一样，


    public int update(String statement, Object[] args, FieldType[] argFieldTypes) throws SQLException {
        return this.update(statement, args, argFieldTypes, "updated");
    } 

     private int update(String statement, Object[] args, FieldType[] argFieldTypes, String label) throws SQLException {
        SQLiteStatement stmt = null;
        ...
       
        stmt = this.db.compileStatement(statement);
        this.bindArgs(stmt, args, argFieldTypes);
        stmt.execute();
        
        int result;
         
        stmt = this.db.compileStatement("SELECT CHANGES()");
        result = (int)stmt.simpleQueryForLong();
         ...
         
        return result;
    }


## 删除(Delete)
前面的步骤和Create的差不多，就不重复，不一样的地方在于他是重用了update函数的


    public int delete(String statement, Object[] args, FieldType[] argFieldTypes) throws SQLException {
        return this.update(statement, args, argFieldTypes, "deleted");
    }

---


# 后记

时间限制，今天就先写到这类把。。。
更多内容再做补充

1. 关于CRUD操作的Buildr模式
2. 直接的查询工作。
3. 还有解析注解部分，生成表的工作
4. 各个类之间调用的类关系图等


---
参考资料：

[Ormlite官网](http://ormlite.com/)