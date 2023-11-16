---
title: 如何优雅切换联表查询表格所使用的Schema
date: 2023/11/16 15:21:30
categories:
- [中间件, 动态数据源]
tags:
- 动态数据源
- Schema
- 联表查询切换
---

### 序言

前面提到过通过动态数据源实现多个数据源之间的连接，本以为能够实现我的多版本知识库切换需求。

但是！如果不出意外肯定就要出意外了！

项目中存在大量联表查询语句，例如：

```sql
select *
from t1
         inner join t2 on t1.template_id = t2.id
         inner join t3 on t3.template_id = t2.id;
```

对于知识库版本的切换需求，`t2`、`t3` 是知识库中的表，它们需要根据知识库的更新而使用不同版本的表，而 `t1` 则是项目中的主表（可以理解为作业表）。

如果使用动态数据源的方式来实现这个需求，因为动态数据源切换连接时整个 `sql` 所处的环境都会被切换，所以就需要将项目中所有这种联表查询给拆出来，容易遗漏不说，还十分麻烦。

通过多数据源来实现知识版本库的切换行不通，那我们可以尝试通过多 `Schema` 来实现，因为 `SQL` 是可以跨 `Schema` 执行的。

所以最终的方案主体逻辑为：

- 客户端新建作业时绑定知识库版本
- 客户端更新知识库时，新建一个 `Schema` 保存新版本的知识库表
- 执行 `SQL` 时根据作业 `id` 获取对应知识库版本，将查询表替换为指定 `Schema` 中的表



### 服务端调整

相较于原来备份时的 `TRUNCATE TABLE` 命令，调整为服务端给当前知识库版本打标签时，备份出当前知识库表和数据，并添加创建 `Schema` 命令。

#### 组装SQL

```sql
@Slf4j
@Component
@RequiredArgsConstructor
public class LibraryDumpHandler {
    private final Environment environment;
    private final DcasProperties properties;

    // 核心线程数
    private static final Integer CORE_THREADS = Runtime.getRuntime().availableProcessors() + 1;
    private final ReentrantLock mainLock = new ReentrantLock();

    public String dump(String version) {
        mainLock.lock();
        ExecutorService executorService = new ThreadPoolExecutor(
                CORE_THREADS, CORE_THREADS,
                0,
                TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<>(100),
                new ThreadFactoryBuilder().setNameFormat("dump-async-name-%d").setDaemon(true).build());
        try {
            Set<TableBean> tableBeans = buildDumpTables();
            final DataSource dataSource = getDataSource();
            Assert.notEmpty(tableBeans);
            // 创建异步任务并使用自定义线程池执行
            List<CompletableFuture<StrBuilder>> futureList = tableBeans.stream().map(bean ->
                    CompletableFuture.supplyAsync(() -> tableDump(dataSource, bean, version), executorService)
            ).collect(Collectors.toList());
            CompletableFuture<Void> allFutures = CompletableFuture.allOf(futureList.toArray(new CompletableFuture[0]));
            // 等待所有异步任务完成
            allFutures.join();
            // 合并所有异步任务结果
            CompletableFuture<StrBuilder> mergedFuture = allFutures.thenApply(voidResult -> {
                StrBuilder mergedResult = StrBuilder.create();
                for (CompletableFuture<StrBuilder> future : futureList) {
                    try {
                        mergedResult.append(future.get());
                    } catch (InterruptedException | ExecutionException e) {
                        log.error(e.getMessage());
                    }
                }
                return mergedResult;
            });
            StrBuilder tableSqlBuilder = mergedFuture.get();
            // 创建schema sql
            StrBuilder schemaSqlBuilder = schemaBuilder(version);
            StrBuilder sqlBuilder = schemaSqlBuilder.append(StrUtil.LF).append(StrUtil.LF).append(tableSqlBuilder);
            // 导出文件
            String path = properties.getDumpPath();
            FileUtil.mkdir(path);
            String fileName = String.format("contentLibrary-%s.sql", version);
            String absolutePath = Func.wrapFilePath(path) + fileName;
            File file = new File(absolutePath);
            FileWriter sqlFile = FileWriter.create(file);
            sqlFile.write(sqlBuilder.toString());

            // 前面的步骤正常执行再打包图片、sql权限查询文件
            CompletableFuture.supplyAsync(() -> zipFolder(properties.getSqlPath(), properties.getDumpPath(),"sql.zip"), executorService);
            CompletableFuture.supplyAsync(() -> zipFolder(properties.getImagePath(), properties.getDumpPath(),"images.zip"), executorService);

            return absolutePath;
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            executorService.shutdown();
            mainLock.unlock();
        }
    }

    /**
     * 打包文件
     *
     * @param targetPath 需要打包的文件目录
     * @param destPath 压缩文件存放路径
     * @param zipName 压缩文件名
     */
    private String zipFolder(String targetPath, String destPath, String zipName) {
        FileUtil.mkdir(targetPath);
        log.info("[知识库备份] 开始备份文件夹，path:{}", targetPath);
        try (FileOutputStream fos = new FileOutputStream(destPath + StrUtil.SLASH + zipName);
             ZipOutputStream zos = new ZipOutputStream(fos)) {
            // addTemplate files to the ZIP file
            File directory = new File(targetPath);
            File[] files = directory.listFiles();
            if (Objects.isNull(files) || files.length == 0) {
                log.info("[知识库备份] 无需备份文件，targetPath:{}", targetPath);
                return zipName;
            }
            for (File file : files) {
                if (!file.isDirectory()) {
                    ZipEntry zipEntry = new ZipEntry(file.getName());
                    zos.putNextEntry(zipEntry);

                    FileInputStream fis = new FileInputStream(file);
                    byte[] buffer = new byte[4096];
                    int length;
                    while ((length = fis.read(buffer)) > 0) {
                        zos.write(buffer, 0, length);
                    }
                    fis.close();
                    zos.closeEntry();
                    log.info("[知识库备份] 文件{}导出成功", file.getName());
                }
            }
        } catch (IOException ex) {
           log.error("Error creating ZIP file: {}", ex.getMessage());
        }
        return zipName;
    }

    // 创建 Schema
    private StrBuilder schemaBuilder(String schema) {
        StrBuilder sqlBuilder = StrBuilder.create();
        sqlBuilder.append("CREATE SCHEMA IF NOT EXISTS ").append("\"").append(schema).append("\"").append(CommonConst.SEMICOLON);
        return sqlBuilder;
    }

    private StrBuilder tableDump(DataSource ds, TableBean table, String schema) {
        Db db = DbUtil.use(ds);
        StrBuilder sqlBuilder = StrBuilder.create();
        try {
            String tableName = table.getTableName();
            if (table.isNeedTruncate()) {
                sqlBuilder.append("TRUNCATE ").append(schema).append(StrUtil.DOT).append(tableName);
                sqlBuilder.append(CommonConst.SEMICOLON + StrUtil.LF);
            }
            if (table.isNeedCreate()) {
                // 这里注意，由于项目使用的是Pgsql，没有MySQL那种show create table命令，所以自定义了一个命令根据表名获取建表语句
                Entity createEntity = db.queryOne("SELECT show_create_table('" + tableName + "')");
                String showCreateTable = (String) createEntity.get("show_create_table");
                String oldTable = "\"" + tableName + "\"";
                String newTable = "\"" + schema + "\".\"" + tableName + "\"";
                String createTableSql = StrUtil.replace(showCreateTable, oldTable, newTable);
                sqlBuilder.append(createTableSql);
                sqlBuilder.append(StrUtil.LF);
            }
            if (table.isNeedInsert()) {
                List<Entity> dataEntities = db.query("SELECT * FROM " + tableName);
                for (Entity dataEntity : dataEntities) {
                    StrBuilder field = StrBuilder.create();
                    StrBuilder data = StrBuilder.create();
                    dataEntity.forEach((fieldName, fieldValue) -> {
                        String valueStr = StrUtil.toStringOrNull(fieldValue);
                        field.append(fieldName).append(", ");
                        if (ObjectUtil.isNotNull(valueStr)) {
                            // 值包含 ' 转义处理
                            valueStr = StrUtil.replace(valueStr, "'", "''");
                            data.append("'").append(valueStr).append("'");
                        } else {
                            data.append("NULL");
                        }
                        data.append(", ");
                    });

                    sqlBuilder.append("INSERT INTO ").append("\"").append(schema).append("\"").append(StrUtil.DOT).append(tableName).append("(");
                    // 去掉最后的逗号和空格
                    String fieldStr = field.subString(0, field.length() - 2);
                    sqlBuilder.append(fieldStr);
                    sqlBuilder.append(") VALUES (");
                    String dataStr = data.subString(0, data.length() - 2);
                    sqlBuilder.append(dataStr);
                    sqlBuilder.append(");");
                    sqlBuilder.append(StrUtil.LF);
                }
            }
            sqlBuilder.append(StrUtil.LF);
            sqlBuilder.append(StrUtil.LF);
            sqlBuilder.append(StrUtil.LF);
        } catch (SQLException e) {
            log.error("数据表{}导出失败，msg:{}", table.getTableName(), e.getMessage());
            throw new RuntimeException(e);
        }
        log.debug("数据表{}导出成功", table.getTableName());
        return sqlBuilder;
    }

    // 获取系统数据源
    private DataSource getDataSource() {
        DataSource ds;
        Assert.notNull(properties);
        try {
            // 优先使用db.setting配置
            ds = DSFactory.get();
        } catch (Exception e) {
            // 通过自己的配置创建数据源连接池
            Setting setting = new Setting();
            setting.put("url", environment.getProperty("spring.datasource.url"));
            setting.put("username", environment.getProperty("spring.datasource.username"));
            setting.put("password", environment.getProperty("spring.datasource.password"));
            try {
                DSFactory dsFactory = DSFactory.create(setting);
                // 设置全局数据源工厂，下次获取无需重新创建
                DSFactory.setCurrentDSFactory(dsFactory);
                return DSFactory.get();
            } catch (Exception ex) {
                log.error("通过配置创建数据源失败");
                throw new RuntimeException(ex);
            }
        }
        return ds;
    }

    // 从项目配置文件中获取需要备份的表封装成Bean
    private Set<TableBean> buildDumpTables() {
        SqlDumpProperties sqlDumpProperties = properties.getSqlDump();
        Assert.notNull(sqlDumpProperties);
        return Arrays.stream(sqlDumpProperties.getTables().split(StrUtil.COMMA)).map(table ->
            TableBean.builder()
                    .tableName(table.trim())
                    .needCreate(sqlDumpProperties.isCreate())
                    .needInsert(sqlDumpProperties.isInsert())
                    .needTruncate(sqlDumpProperties.isTruncate())
                    .build()
        ).collect(Collectors.toSet());
    }
}
```



#### 自定义查询PGSQL建表方法

```sql
create function show_create_table(in_table_name character varying) returns text
    language plpgsql
as
$$
DECLARE
    -- the ddl we're building
    v_table_ddl text;

    -- data about the target table
    v_table_oid int;

    v_table_type char;
    v_partition_key varchar;
    v_namespace varchar;
    v_table_comment varchar;

    -- records for looping
    v_column_record record;
    v_constraint_record record;
    v_index_record record;
    v_column_comment_record record;
    v_index_comment_record record;
    v_constraint_comment_record record;
BEGIN
    -- grab the oid of the table;
SELECT c.oid, c.relkind, n.nspname INTO v_table_oid, v_table_type, v_namespace
FROM pg_catalog.pg_class c
         LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind in ('r', 'p')
  AND c.relname = in_table_name -- the table name
  AND pg_catalog.pg_table_is_visible(c.oid); -- the schema

-- throw an error if table was not found
IF (v_table_oid IS NULL) THEN
        RAISE EXCEPTION 'table does not exist';
END IF;

    -- start the create definition
    v_table_ddl := 'CREATE TABLE "' || in_table_name || '" (' || E'\n';

    -- define all of the columns in the table;
FOR v_column_record IN
SELECT
    c.column_name,
    c.data_type,
    c.character_maximum_length,
    c.is_nullable,
    c.column_default
FROM information_schema.columns c
WHERE table_name = in_table_name and table_schema = v_namespace
ORDER BY ordinal_position
    LOOP
            v_table_ddl := v_table_ddl || '  ' -- note: two char spacer to start, to indent the column
                               || '"' || v_column_record.column_name || '" '
                               || v_column_record.data_type || CASE WHEN v_column_record.character_maximum_length IS NOT NULL THEN ('(' || v_column_record.character_maximum_length || ')') ELSE '' END || ' '
                               || CASE WHEN v_column_record.is_nullable = 'NO' THEN 'NOT NULL' ELSE 'NULL' END
                               || CASE WHEN v_column_record.column_default IS NOT null THEN (' DEFAULT ' || v_column_record.column_default) ELSE '' END
                               || ',' || E'\n';
END LOOP;

    -- define all the constraints in the;
FOR v_constraint_record IN
SELECT
    con.conname as constraint_name,
    con.contype as constraint_type,
    CASE
        WHEN con.contype = 'p' THEN 1 -- primary key constraint
        WHEN con.contype = 'u' THEN 2 -- unique constraint
        WHEN con.contype = 'f' THEN 3 -- foreign key constraint
        WHEN con.contype = 'c' THEN 4
        ELSE 5
        END as type_rank,
    pg_get_constraintdef(con.oid) as constraint_definition
FROM pg_catalog.pg_constraint con
         JOIN pg_catalog.pg_class rel ON rel.oid = con.conrelid
         JOIN pg_catalog.pg_namespace nsp ON nsp.oid = connamespace
WHERE rel.relname = in_table_name
  AND pg_catalog.pg_table_is_visible(rel.oid)
ORDER BY type_rank
    LOOP
            IF v_constraint_record.constraint_type = 'p' THEN
                v_table_ddl := v_table_ddl || '  '
                                   || v_constraint_record.constraint_definition
                                   || ',' || E'\n';
ELSE
                v_table_ddl := v_table_ddl || '  ' -- note: two char spacer to start, to indent the column
                                   || 'CONSTRAINT' || ' '
                                   || '"' || v_constraint_record.constraint_name || '" '
                                   || v_constraint_record.constraint_definition
                                   || ',' || E'\n';
END IF;
END LOOP;

    -- drop the last comma before ending the create statement
    v_table_ddl = substr(v_table_ddl, 0, length(v_table_ddl) - 1) || E'\n';

    -- end the create definition
    v_table_ddl := v_table_ddl || ')';

    IF v_table_type = 'p' THEN
SELECT pg_get_partkeydef(v_table_oid) INTO v_partition_key;
IF v_partition_key IS NOT NULL THEN
            v_table_ddl := v_table_ddl || ' PARTITION BY ' || v_partition_key;
END IF;
END IF;

    v_table_ddl := v_table_ddl || ';' || E'\n';

    -- suffix create statement with all of the indexes on the table
FOR v_index_record IN
SELECT regexp_replace(idx.indexdef, ' "?' || idx.schemaname || '"?\.' || '"?' || idx.tablename || '"?', ' "' || idx.tablename || '" ') AS indexdef
FROM pg_indexes idx
         JOIN (
    SELECT ns.nspname, cls.relname
    FROM pg_catalog.pg_class cls
             LEFT JOIN pg_catalog.pg_namespace ns ON ns.oid = cls.relnamespace
    WHERE pg_catalog.pg_table_is_visible(cls.oid)
) t ON idx.schemaname = t.nspname AND idx.tablename = t.relname
WHERE idx.tablename = in_table_name
  AND idx.indexname NOT IN (
    select con.conname
    FROM pg_catalog.pg_constraint con
             JOIN pg_catalog.pg_class rel ON rel.oid = con.conrelid
             JOIN pg_catalog.pg_namespace nsp ON nsp.oid = connamespace
    WHERE rel.relname = in_table_name
      AND pg_catalog.pg_table_is_visible(rel.oid)
)
    LOOP
            v_table_ddl := v_table_ddl
                               || v_index_record.indexdef
                               || ';' || E'\n';
END LOOP;

    -- comment on table
SELECT description INTO v_table_comment
FROM pg_catalog.pg_description
WHERE objoid = v_table_oid AND objsubid = 0;

IF v_table_comment IS NOT NULL THEN
        v_table_ddl := v_table_ddl || 'COMMENT ON TABLE "' || in_table_name || '" IS ''' || replace(v_table_comment, '''', '''''') || ''';' || E'\n';
END IF;

    -- comment on column
FOR v_column_comment_record IN
SELECT col.column_name, d.description
FROM information_schema.columns col
         JOIN pg_catalog.pg_class c ON c.relname = col.table_name
         JOIN pg_catalog.pg_namespace nsp ON nsp.oid = c.relnamespace AND col.table_schema = nsp.nspname
         JOIN pg_catalog.pg_description d ON d.objoid = c.oid AND d.objsubid = col.ordinal_position
WHERE c.oid = v_table_oid
ORDER BY col.ordinal_position
    LOOP
            v_table_ddl := v_table_ddl || 'COMMENT ON COLUMN "' || in_table_name || '"."'
                               || v_column_comment_record.column_name || '" IS '''
                               || replace(v_column_comment_record.description, '''', '''''') || ''';' || E'\n';
END LOOP;

    -- comment on index
FOR v_index_comment_record IN
SELECT c.relname, d.description
FROM pg_catalog.pg_index idx
         JOIN pg_catalog.pg_class c ON idx.indexrelid = c.oid
         JOIN pg_catalog.pg_description d ON idx.indexrelid = d.objoid
WHERE idx.indrelid = v_table_oid
    LOOP
            v_table_ddl := v_table_ddl || 'COMMENT ON INDEX "'
                               || v_index_comment_record.relname || '" IS '''
                               || replace(v_index_comment_record.description, '''', '''''') || ''';' || E'\n';
END LOOP;

    -- comment on constraint
FOR v_constraint_comment_record IN
SELECT
    con.conname,
    pg_description.description
FROM pg_catalog.pg_constraint con
         JOIN pg_catalog.pg_class rel ON rel.oid = con.conrelid
         JOIN pg_catalog.pg_namespace nsp ON nsp.oid = connamespace
         JOIN pg_catalog.pg_description ON pg_description.objoid = con.oid
WHERE rel.oid = v_table_oid
    LOOP
            v_table_ddl := v_table_ddl || 'COMMENT ON CONSTRAINT "'
                               || v_constraint_comment_record.conname || '" ON "' || in_table_name || '" IS '''
                               || replace(v_constraint_comment_record.description, '''', '''''') || ''';' || E'\n';
END LOOP;

    -- return the ddl
RETURN v_table_ddl;
END
$$;

-- 替换为你的用户
alter function show_create_table(varchar) owner to your_user;
```



### 客户端调整

#### 上下文

内部维护了一个栈结构保存知识库版本用于处理 `Schema` 切换的嵌套情况，例如：

1. 在某个业务逻辑中需要切换到`Schema A`。
2. 在这个业务逻辑的某个方法内部调用了另一个方法，这个方法需要切换到`Schema B`。

```java
public final class DynamicSchemaContextHolder {

    // 保存知识库版本
    private static final ThreadLocal<Deque<String>> SCHEME_KEY_HOLDER = TransmittableThreadLocal.withInitial(ArrayDeque::new);

    private DynamicSchemaContextHolder() {}

    public static String peek() {
        return SCHEME_KEY_HOLDER.get().peek();
    }

    public static void push(String ds) {
        String schemeKeyStr = StrUtil.isEmpty(ds) ? StrUtil.EMPTY : ds;
        SCHEME_KEY_HOLDER.get().push(schemeKeyStr);
    }

    public static void poll() {
        Deque<String> deque = SCHEME_KEY_HOLDER.get();
        deque.poll();
        if (deque.isEmpty()) {
            SCHEME_KEY_HOLDER.remove();
        }
    }

    public static void clear() {
        SCHEME_KEY_HOLDER.remove();
    }
}
```



#### 管理器

使用一个 `Manager` 管理以下属性：

- 默认 `Schema` ，根据版本号无法找到对应 `Schema` 时切换到默认 `Schema`
- 知识库表，动态切换 `Schema` 时需要判断表是否存在该集合中，存在即替换，否则默认使用`public Schema`
- 知识库版本对应的 `Schema` 名称
- 知识库版本对应的作业（一对多）

这个类的主要作用是更新知识库时，将添加的知识库和Schema保存起来，拦截器中根据版本号获取`Schema` 等。

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class DynamicSchemaManager {
    // 默认schema
    private static final String DEFAULT_SCHEMA = "public";

    // 最新版本知识库
    private static String LATEST_VERSION = "public";
    // 保存知识库相关的表，用于在sql中判断是否需要切换schema，存在此集合中的表需要切换schema
    private static final Set<String> SYNC_TABLE_NAMES = new HashSet<>();
    // key: version, value: schemaName
    private static final Map<String, String> SCHEME_KEY_MAP = Maps.newConcurrentMap();
    // key:version, value:operationId
    private static final Multimap<String, String> VERSION_WORK = Multimaps.newMultimap(Maps.newHashMap(), HashSet::new);

    private final CoOperationMapper coOperationMapper;
    private final LibrarySyncConfigMapper librarySyncConfigMapper;
    private final LibrarySyncTablesMapper librarySyncTablesMapper;

    @PostConstruct
    public void init() {
        List<LibrarySyncConfig> configs = librarySyncConfigMapper.selectList(new QueryWrapper<>());
        if (CollUtil.isNotEmpty(configs))
            configs.forEach(config -> {
                SCHEME_KEY_MAP.put(config.getVersion(), config.getSchemaName());
                log.info("dynamic-schema - add a schema named [{}] success", config.getSchemaName());
                if (config.getIsLatest()) {
                    LATEST_VERSION = config.getVersion();
                    log.info("dynamic-schema - set the latest schema named [{}] success", config.getSchemaName());
                }
            });
        Set<String> librarySyncTables = librarySyncTablesMapper.selectSyncTableNames();
        SYNC_TABLE_NAMES.addAll(librarySyncTables);
        List<Pair<String, String>> pairs = coOperationMapper.selectLibraryVersion();
        if (CollUtil.isNotEmpty(pairs))
            pairs.forEach(pair -> VERSION_WORK.put(pair.getKey(), pair.getValue()));
        log.info("dynamic-schema - init success");
    }

    /*
     * 根据版本号获取schema
     */
    public static String getSchema(String version) {
        if (StrUtil.isEmpty(version)) {
            return determineDefaultSchema();
        }
        String schemaName = SCHEME_KEY_MAP.get(version);
        if (StrUtil.isEmpty(schemaName)) {
            schemaName = determineDefaultSchema();
            log.warn("dynamic-schema - can not find the schema named [{}], switch to the default schema [{}]", version, schemaName);
        }
        log.debug("dynamic-schema - switch to the schema named [{}]", schemaName);
        return schemaName;
    }

    /*
     * 选择默认schema
     */
    private static String determineDefaultSchema() {
        return DEFAULT_SCHEMA;
    }

    /*
     * 添加schema
     */
    public static synchronized void addSchema(String version, String schemaName) {
        SCHEME_KEY_MAP.put(version, schemaName);
        log.info("dynamic-schema - add a schema named [{}] success", schemaName);
        LATEST_VERSION = version;
        log.info("dynamic-schema - set the latest schema named [{}] success", schemaName);
    }

    /*
     * 删除schema
     */
    public static void removeSchema(String version) {
        SCHEME_KEY_MAP.remove(version);
        log.info("dynamic-schema - remove a schema named [{}] success", version);
    }

    /*
     * 判断sql中是否包含需要切换schema的表
     */
    public static boolean isContainsSyncTable(String sql) {
        return SYNC_TABLE_NAMES.stream().map(String::toLowerCase).anyMatch(sql::contains);
    }

    /*
     * 获取需要切换schema的表
     */
    public static Set<String> getSyncTableNames() {
        return SYNC_TABLE_NAMES;
    }

    /*
     * 获取版本号
     */
    public static String getVersion(String operationId) {
        if (!VERSION_WORK.containsValue(operationId))
            return null;
        for (Map.Entry<String, Collection<String>> entry : VERSION_WORK.asMap().entrySet()) {
            Collection<String> values = entry.getValue();
            if (values.contains(operationId))
                return entry.getKey();
        }
        return null;
    }

    /*
     * 获取最新版本号
     */
    public static synchronized String getLatestVersion() {
        return LATEST_VERSION;
    }

    /*
     * 添加版本号
     */
    public static synchronized void addVersion(String version, String operationId) {
        VERSION_WORK.put(version, operationId);
    }

    /*
     * 删除版本号中的作业id
     */
    public static synchronized void removeVersionWork(String operationId) {
        if (!VERSION_WORK.containsValue(operationId))
            return;
        for (Map.Entry<String, Collection<String>> entry : VERSION_WORK.asMap().entrySet()) {
            Collection<String> values = entry.getValue();
            if (values.contains(operationId)) {
                values.remove(operationId);
                break;
            }
        }
    }
}
```



### 自定义切换注解

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface SchemaSwitch {

    // 指定包含作业id的类，作业id用于查找知识库版本
    Class<?> value() default LatestVersion.class;
}
```



### 切面

获取注解上配置的 `Class`，因为该`Class` 配置为包含作业 `id` 的类。

通过反射获取作业 `id`，通过作业 `id` 获取对应的知识库版本，如果版本号不为空，则放入到当前线程的本地线程变量中。

这里注意方法执行完之后要调用 `clear()` 清理，防止内存溢出。

```java
@Aspect
@Component
public class SchemaAspect {

    @Pointcut("@annotation(com.dcas.common.annotation.SchemaSwitch)")
    public void switchSchema() {
    }

    @Around("switchSchema()")
    public Object around(ProceedingJoinPoint pointcut) throws Throwable {
        String version = null;
        String operationId = null;
        MethodSignature signature = (MethodSignature)pointcut.getSignature();
        SchemaSwitch annotation = signature.getMethod().getAnnotation(SchemaSwitch.class);
        Class<?> clazz = annotation.value();
        if (clazz == LatestVersion.class) {
            // 默认切换最新版本知识库
            version = DynamicSchemaManager.getLatestVersion();
        } else {
            Object[] args = pointcut.getArgs();
            for (Object arg : args) {
                if (arg.getClass() != clazz)
                    continue;
                Object obj = JSONObject.parseObject(JSON.toJSONString(arg), clazz);
                if (clazz == String.class || clazz == Integer.class) {
                    operationId = obj.toString();
                } else {
                    operationId = ReflectUtil.getFieldValue(obj, "operationId").toString();
                }
                if (StrUtil.isEmpty(operationId))
                    continue;
                version = DynamicSchemaManager.getVersion(operationId);
                break;
            }
        }
        if (StrUtil.isNotEmpty(version))
            DynamicSchemaContextHolder.push(version);
        try {
            return pointcut.proceed();
        } finally {
            DynamicSchemaContextHolder.clear();
        }
    }
}
```



### 动态替换

前面所有的操作都是为了接下来的这一步做铺垫，要动态切换知识库相关表所在的`Schema`，我们现在已经将知识库版本放入到了本地线程变量，只需要通过 `DynamicSchemaManager` 的 `getSchema(String version)` 方法获取版本对应的 `schema` 名称，然后将名称设置到 `sql `的表名前面。

#### 占位符

很容易想到的一种方法就是在 `xxxMapper.xml` 文件中需要动态切换 `Schema` 的表名前面加上占位符，然后将 `Schema` 名称通过参数传入。

例如：

```java
T methodA(@Param("schema") String schema);
```

```sql
select *
from t1
         inner join #{schema}.t2 on t1.template_id = t2.id
         inner join #{schema} on t3.template_id = t2.id;
```

额。。。兄弟，前面都挺好的，怎么到你这这么拉了？

这种方法无疑有大量的改动，而且对于每次新增方法也要同步修改，别说你们我都忍不了。

难道还有更好的办法？



### 拦截器

`org.apache.ibatis.plugin.Interceptor` 是 `MyBatis` 提供的拦截器接口，可以用于在 `MyBatis` 执行 `SQL` 语句的各个阶段进行拦截和干预。

通过实现这个接口，我们可以在 `SQL` 语句执行前后，以及执行期间做一些额外的处理。

由于我的项目中使用了 `MybatisPlus`, 已经实现了拦截器接口，另外做了一层封装，提供了`InnerInterceptor` 接口，只需要实现这个接口就可以对待执行的 `SQL` 进行一些修改逻辑。

![image-20231116175130355](https://blog.seeyourface.cn/blog/image-20231116175130355.png)

```java
public class SchemaSelectInterceptor implements InnerInterceptor {

    @Override
    public void beforeQuery(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
        String sql = boundSql.getSql();
        // 简单判断sql中是否包含涉及的知识库表，不包含的话就不用解析了
        if (!isContainsSyncTable(sql)) {
            return;
        }
        String version = DynamicSchemaContextHolder.peek();
        String schema = DynamicSchemaManager.getSchema(version);
        PGSQLStatementParser parser = new PGSQLStatementParser(sql);
        SQLStatement sqlStatement = parser.parseStatement();
        // 访问器
        SchemaSwitchVisitorAdapter visitor = new SchemaSwitchVisitorAdapter(schema, DynamicSchemaManager.getSyncTableNames());
        sqlStatement.accept(visitor);
        PluginUtils.mpBoundSql(boundSql).sql(SQLUtils.toSQLString(sqlStatement, DbType.postgresql));
    }

    private boolean isContainsSyncTable(String sql) {
        return DynamicSchemaManager.isContainsSyncTable(sql.toLowerCase());
    }
}
```

我们现在能拦截执行前的 `SQL`，现在剩下的首要任务就是将需要替换`schema`的表正确替换，考虑到有一些表有很相似的名称，不能简单进行 `replace`。

这里我使用了阿里 `Druid` 数据源提供的 `SQL` 解析器 `PGSQLStatementParser`，我们先大概判断有没有需要替换的表，没有的话就没必要进行解析了。

然后自定义一个 `AST` 节点访问器 `SchemaSwitchVisitorAdapter`。

```java
@AllArgsConstructor
public class SchemaSwitchVisitorAdapter extends PGASTVisitorAdapter {

    private static final String DOUBLE_QUOTE = "\"";
    private final String schema;
    private final Set<String> syncTables;

    @Override
    public boolean visit(SQLExprTableSource x) {
        String tableName = StrUtil.removeAll(x.getExpr().toString(), DOUBLE_QUOTE);
        if (syncTables.stream().anyMatch(tableName::equalsIgnoreCase))
            x.setExpr(DOUBLE_QUOTE + schema + DOUBLE_QUOTE + StrUtil.DOT + tableName);
        return true;
    }
}
```

通过构造函数传入待替换的 `Schema` 和知识库的表，判断当前访问的表是否存在于知识库表中，如果是的话就在前面加上要替换的 `Schema`。

通过这种操作，我们无需修改原先的任何 `SQL`。

### 最终效果

```tex
18:03:23.208 [http-nio-8080-exec-55] DEBUG c.d.s.m.DynamicSchemaManager - [getSchema,80] - dynamic-schema - switch to the schema named [20231115171241]
18:03:23.213 [http-nio-8080-exec-55] DEBUG c.d.s.m.A.qryByTemplateType - [debug,137] - ==>  Preparing: SELECT t1."name", t4.bp_code, t4."stage", t4."process", t4.dimension , t4."content", t4."level", t6.id AS itemId FROM "20231115171241".analysis_template t1 INNER JOIN "20231115171241".standard_template_relevance t2 ON t1."id" = t2.template_id INNER JOIN "20231115171241".standard t3 ON t2.standard_id = t3."id" AND t3."enable" = true INNER JOIN "20231115171241".standard_item t4 ON t3."id" = t4.standard_id LEFT JOIN "20231115171241".clause_item_relevance t5 ON t4."id" = t5.clause_id LEFT JOIN "20231115171241".item t6 ON t5.item_id = t6.id AND t6.status = 0 WHERE t1."type" = ? AND t4."level" = ?
18:03:23.213 [http-nio-8080-exec-55] DEBUG c.d.s.m.A.qryByTemplateType - [debug,137] - ==> Parameters: 1(Integer), 3(Integer)
18:03:23.255 [http-nio-8080-exec-55] DEBUG c.d.s.m.A.qryByTemplateType - [debug,137] - <==      Total: 385
```





