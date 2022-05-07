# Mybatis源码

## 1. Mybatis优缺点

JDBC的弊端:
- 底层没有用到连接池，频繁的创建和关闭，消耗资源
- 非对象的方式传递参，sql需要修改的时候需要重新编译，不利于维护
- 预编译的时候变量位置使用123，不利于维护
- result结果集需要硬编码

Mybatis优点：

1. sql灵活，支持动态sql
2. sql链接托管，无需手动关闭，池化技术
3. 与spring很好的集成

Mybatis缺点：
1. 因为sql灵活，编写量大，需要sql基础
2. 依赖数据库，移植比较差 相对于`Hibernate`

总结：
与Hibernate比较，Hibernate的对象/关系映射能力强，数据库无关性好，对于关系模型要求高的软件，如果用 hibernate 开发可以节省很多代码，提高效率

## 2. 调用sql的三种方式，优先级是什么

namespace的xml调用
```java
UmsAdmin umsAdmin = sqlSession.selectOne("com.xuzhihao.mbg.mapper.UmsAdminMapper.selectByPrimaryKey", 1L);
```

Mapper调用
```java
UmsAdminMapper umsAdminMapper = sqlSession.getMapper(UmsAdminMapper.class);
```

Annotation注解调用
```java
UmsAdminAnnotationMapper umsAdminAnnotationMapper = sqlSession.getMapper(UmsAdminAnnotationMapper.class);
```

## 3. mybaits的一级缓存、二级缓存

Spring只有在开启了事务之后，在同一个事务里的SqlSession会被缓存起来，同一个事务中，多次查询是可以命中缓存的

```java
private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
       //获取SqlSession
      SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
      try {
        //反射调用真正的处理方法
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          //提交数据
          sqlSession.commit(true);
        }
        //返回查询的数据
        return result;
      } catch (Throwable t) {
        //。。。。忽略不必要代码
      } finally {
        if (sqlSession != null) {
            //关闭SqlSession的连接
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```

MyBatis 一级缓存最大的共享范围就是一个SqlSession内部，那么如果多个 SqlSession 需要共享缓存，则需要开启二级缓存

```xml
<settings>
	<setting name = "cacheEnabled" value = "true" />
</settings>
```

当二级缓存开启后，同一个命名空间(namespace) 所有的操作语句，都影响着一个共同的 cache，也就是二级缓存被多个 SqlSession 共享，是一个全局的变量


## 4. 插件扩展原理和经典案例

### 4.1 慢sql日志

```java
package com.xuzhihao.shop.admin.component;

import java.sql.Date;
import java.text.SimpleDateFormat;
import java.util.List;
import java.util.Properties;

import org.apache.ibatis.cache.CacheKey;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.ParameterMapping;
import org.apache.ibatis.mapping.ParameterMode;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.apache.ibatis.plugin.Signature;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import org.apache.ibatis.type.TypeHandlerRegistry;

import lombok.extern.slf4j.Slf4j;

/**
 * 慢sql分析插件
 * 
 * @author Administrator
 *
 */
@Intercepts({
		@Signature(type = Executor.class, method = "query", args = { MappedStatement.class, Object.class,
				RowBounds.class, ResultHandler.class }),
		@Signature(type = Executor.class, method = "query", args = { MappedStatement.class, Object.class,
				RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class }),
		@Signature(type = Executor.class, method = "update", args = { MappedStatement.class, Object.class }) })
@Slf4j
public class SqlMonitorPlugin implements Interceptor {
	@Override
	public Object intercept(Invocation invocation) throws Throwable {
		MappedStatement mappedStatement = (MappedStatement) invocation.getArgs()[0];
		Object parameter = null;
		if (invocation.getArgs().length > 1) {
			parameter = invocation.getArgs()[1];
		}
		String sqlId = mappedStatement.getId();
		BoundSql boundSql = mappedStatement.getBoundSql(parameter);
		Configuration configuration = mappedStatement.getConfiguration();
		long startTime = System.currentTimeMillis();
		Object result = null;
		try {
			result = invocation.proceed();
		} finally {
			long timing = System.currentTimeMillis() - startTime;
			String sql = getSql(boundSql, parameter, configuration);
			log.info("执行sql耗时：{}ms -id：{} -sql:{}", timing, sqlId, sql);
		}
		return result;
	}

	private String getSql(BoundSql boundSql, Object paramerObject, Configuration configuration) {
		String sql = boundSql.getSql().replaceAll("[\\s]+", " ");
		List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
		TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
		if (parameterMappings != null) {
			for (int i = 0; i < parameterMappings.size(); i++) {
				ParameterMapping parameterMapping = parameterMappings.get(i);
				if (parameterMapping.getMode() != ParameterMode.OUT) {
					Object value;
					String propertyName = parameterMapping.getProperty();
					if (boundSql.hasAdditionalParameter(propertyName)) {
						value = boundSql.getAdditionalParameter(propertyName);
					} else if (paramerObject == null) {
						value = null;
					} else if (typeHandlerRegistry.hasTypeHandler(paramerObject.getClass())) {
						value = paramerObject;
					} else {
						MetaObject metaObject = configuration.newMetaObject(paramerObject);
						value = metaObject.getValue(propertyName);
					}
					sql = replacePlaceholder(sql, value);
				}
			}
		}
		return sql;
	}

	/**
	 * 填充占位符?
	 *
	 * @param sql
	 * @param parameterObject
	 * @return
	 */
	private String replacePlaceholder(String sql, Object parameterObject) {
		String result;
		if (parameterObject instanceof String) {
			result = "'" + parameterObject.toString() + "'";
		} else if (parameterObject instanceof Date) {
			SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
			result = "'" + simpleDateFormat.format(parameterObject) + "'";
		} else {
			result = parameterObject.toString();
		}
		return sql.replaceFirst("\\?", result);
	}

	@Override
	public Object plugin(Object target) {
		if (target instanceof Executor) {
			return Plugin.wrap(target, this);
		}
		return target;
	}

	@Override
	public void setProperties(Properties properties) {

	}

}
```

### 4.2 乐观锁

```xml
<plugins>
		<plugin interceptor="com.xuzhihao.interceptor.OptimisticLocker">
			<property name="versionColumn" value="version" /> <!-- 数据库的列名 -->
			<property name="versionField" value="version" /> <!-- java字段名 -->
		</plugin>
		<plugin interceptor="com.xuzhihao.interceptor.FieldEncryptInterceptor" />
		<plugin interceptor="com.xuzhihao.interceptor.SqlMonitorPlugin" />
	</plugins>
```

```java
package com.xuzhihao.interceptor;

import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.sql.Connection;
import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.Properties;

import org.apache.ibatis.binding.BindingException;
import org.apache.ibatis.binding.MapperMethod;
import org.apache.ibatis.executor.parameter.ParameterHandler;
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.apache.ibatis.plugin.Signature;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.reflection.SystemMetaObject;
import org.apache.log4j.Logger;

import com.xuzhihao.Appliaction;
import com.xuzhihao.interceptor.locker.Cache;
import com.xuzhihao.interceptor.locker.LocalVersionLockerCache;
import com.xuzhihao.interceptor.locker.VersionLocker;
import com.xuzhihao.interceptor.locker.VersionLockerCache;
import com.xuzhihao.interceptor.locker.Cache.MethodSignature;

import net.sf.jsqlparser.expression.Expression;
import net.sf.jsqlparser.expression.LongValue;
import net.sf.jsqlparser.expression.operators.arithmetic.Addition;
import net.sf.jsqlparser.expression.operators.conditional.AndExpression;
import net.sf.jsqlparser.expression.operators.relational.EqualsTo;
import net.sf.jsqlparser.parser.CCJSqlParserUtil;
import net.sf.jsqlparser.schema.Column;
import net.sf.jsqlparser.statement.Statement;
import net.sf.jsqlparser.statement.update.Update;

/**
 * MyBatis乐观锁
 * 
 * @author Administrator
 *
 */
@Intercepts({
		@Signature(type = StatementHandler.class, method = "prepare", args = { Connection.class, Integer.class }) })
public class OptimisticLocker implements Interceptor {

	private static Logger log = Logger.getLogger(Appliaction.class);

	private static VersionLocker trueLocker;
	private static VersionLocker falseLocker;

	static {
		try {
			trueLocker = OptimisticLocker.class.getDeclaredMethod("versionValueTrue")
					.getAnnotation(VersionLocker.class);
			falseLocker = OptimisticLocker.class.getDeclaredMethod("versionValueFalse")
					.getAnnotation(VersionLocker.class);
		} catch (NoSuchMethodException | SecurityException e) {
			throw new RuntimeException("The plugin init faild.", e);
		}
	}

	private Properties props = null;
	private VersionLockerCache versionLockerCache = new LocalVersionLockerCache();

	@VersionLocker(true)
	private void versionValueTrue() {
	}

	@VersionLocker(false)
	private void versionValueFalse() {
	}

	@Override
	public Object intercept(Invocation invocation) throws Exception {
		String versionColumn;
		String versionField;
		if (null == props || props.isEmpty()) {
			versionColumn = "version";
			versionField = "version";
		} else {
			versionColumn = props.getProperty("versionColumn", "version");
			versionField = props.getProperty("versionField", "version");
		}
		String interceptMethod = invocation.getMethod().getName();
		if (!"prepare".equals(interceptMethod)) {
			return invocation.proceed();
		}
		StatementHandler handler = (StatementHandler) processTarget(invocation.getTarget());
		MetaObject metaObject = SystemMetaObject.forObject(handler);
		MappedStatement ms = (MappedStatement) metaObject.getValue("delegate.mappedStatement");
		SqlCommandType sqlCmdType = ms.getSqlCommandType();
		if (sqlCmdType != SqlCommandType.UPDATE) {
			return invocation.proceed();
		}
		BoundSql boundSql = (BoundSql) metaObject.getValue("delegate.boundSql");
		VersionLocker vl = getVersionLocker(ms, boundSql);
		if (null != vl && !vl.value()) {
			return invocation.proceed();
		}
		Object originalVersion = metaObject.getValue("delegate.boundSql.parameterObject." + versionField);
		if (originalVersion == null || Long.parseLong(originalVersion.toString()) <= 0) {
			throw new BindingException("value of version field[" + versionField + "]can not be empty");
		}
		String originalSql = boundSql.getSql();

		log.info("==> originalSql: " + originalSql);

		originalSql = addVersionToSql(originalSql, versionColumn, originalVersion);
		metaObject.setValue("delegate.boundSql.sql", originalSql);

		log.info("==> originalSql after add version: " + originalSql);

		return invocation.proceed();
	}

	private String addVersionToSql(String originalSql, String versionColumnName, Object originalVersion) {
		try {
			Statement stmt = CCJSqlParserUtil.parse(originalSql);
			if (!(stmt instanceof Update)) {
				return originalSql;
			}
			Update update = (Update) stmt;
			if (!contains(update, versionColumnName)) {
				buildVersionExpression(update, versionColumnName);
			}
			Expression where = update.getWhere();
			if (where != null) {
				AndExpression and = new AndExpression(where, buildVersionEquals(versionColumnName, originalVersion));
				update.setWhere(and);
			} else {
				update.setWhere(buildVersionEquals(versionColumnName, originalVersion));
			}
			return stmt.toString();
		} catch (Exception e) {
			e.printStackTrace();
			return originalSql;
		}
	}

	private boolean contains(Update update, String versionColumnName) {
		List<Column> columns = update.getColumns();
		for (Column column : columns) {
			if (column.getColumnName().equalsIgnoreCase(versionColumnName)) {
				return true;
			}
		}
		return false;
	}

	private void buildVersionExpression(Update update, String versionColumnName) {

		List<Column> columns = update.getColumns();
		Column versionColumn = new Column();
		versionColumn.setColumnName(versionColumnName);
		columns.add(versionColumn);

		List<Expression> expressions = update.getExpressions();
		Addition add = new Addition();
		add.setLeftExpression(versionColumn);
		add.setRightExpression(new LongValue(1));
		expressions.add(add);
	}

	private Expression buildVersionEquals(String versionColumnName, Object originalVersion) {
		EqualsTo equal = new EqualsTo();
		Column column = new Column();
		column.setColumnName(versionColumnName);
		equal.setLeftExpression(column);
		LongValue val = new LongValue(originalVersion.toString());
		equal.setRightExpression(val);
		return equal;
	}

	private VersionLocker getVersionLocker(MappedStatement ms, BoundSql boundSql) {

		Class<?>[] paramCls = null;
		Object paramObj = boundSql.getParameterObject();

		/****************** 下面处理参数只能按照下面3个的顺序 ***********************/
		/****************** Process param must order by below ***********************/
		// 1、处理@Param标记的参数
		// 1、Process @Param param
		if (paramObj instanceof MapperMethod.ParamMap<?>) {
			MapperMethod.ParamMap<?> mmp = (MapperMethod.ParamMap<?>) paramObj;
			if (null != mmp && !mmp.isEmpty()) {
				paramCls = new Class<?>[mmp.size() / 2];
				int mmpLen = mmp.size() / 2;
				for (int i = 0; i < mmpLen; i++) {
					Object index = mmp.get("param" + (i + 1));
					paramCls[i] = index.getClass();
					if (List.class.isAssignableFrom(paramCls[i])) {
						return falseLocker;
					}
				}
			}

			// 2、处理Map类型参数
			// 2、Process Map param
		} else if (paramObj instanceof Map) {// 不支持批量
			@SuppressWarnings("rawtypes")
			Map map = (Map) paramObj;
			if (map.get("list") != null || map.get("array") != null) {
				return falseLocker;
			} else {
				paramCls = new Class<?>[] { Map.class };
			}
			// 3、处理POJO实体对象类型的参数
			// 3、Process POJO entity param
		} else {
			paramCls = new Class<?>[] { paramObj.getClass() };
		}

		Cache.MethodSignature vm = new MethodSignature(ms.getId(), paramCls);
		VersionLocker versionLocker = versionLockerCache.getVersionLocker(vm);
		if (null != versionLocker) {
			return versionLocker;
		}

		Class<?> mapper = getMapper(ms);
		if (mapper != null) {
			Method m;
			try {
				m = mapper.getDeclaredMethod(getMapperShortId(ms), paramCls);
			} catch (NoSuchMethodException | SecurityException e) {
				throw new RuntimeException("The Map type param error." + e, e);
			}
			versionLocker = m.getAnnotation(VersionLocker.class);
			if (null == versionLocker) {
				versionLocker = trueLocker;
			}
			if (!versionLockerCache.containMethodSignature(vm)) {
				versionLockerCache.cacheMethod(vm, versionLocker);
			}
			return versionLocker;
		} else {
			throw new RuntimeException("Config info error, maybe you have not config the Mapper interface");
		}
	}

	private Class<?> getMapper(MappedStatement ms) {
		String namespace = getMapperNamespace(ms);
		Collection<Class<?>> mappers = ms.getConfiguration().getMapperRegistry().getMappers();
		for (Class<?> clazz : mappers) {
			if (clazz.getName().equals(namespace)) {
				return clazz;
			}
		}
		return null;
	}

	private String getMapperNamespace(MappedStatement ms) {
		String id = ms.getId();
		int pos = id.lastIndexOf(".");
		return id.substring(0, pos);
	}

	private String getMapperShortId(MappedStatement ms) {
		String id = ms.getId();
		int pos = id.lastIndexOf(".");
		return id.substring(pos + 1);
	}

	@Override
	public Object plugin(Object target) {
		if (target instanceof StatementHandler || target instanceof ParameterHandler) {
			return Plugin.wrap(target, this);
		} else {
			return target;
		}
	}

	@Override
	public void setProperties(Properties properties) {
		if (null != properties && !properties.isEmpty())
			props = properties;
	}

	public static Object processTarget(Object target) {
		if (Proxy.isProxyClass(target.getClass())) {
			MetaObject mo = SystemMetaObject.forObject(target);
			return processTarget(mo.getValue("h.target"));
		}
		return target;
	}
}
```

```java
@VersionLocker()
@SensitiveConverter(value = SensitiveType.UPDATE)
int updateByPrimaryKey(UmsDemo record);
```

### 4.3 数据脱敏

```java
public enum SensitiveType {
	/**
	 * 保存前过滤
	 */
	UPDATE,
	/**
	 * 显示脱敏
	 */
	SELECT,
}
```

```java
package com.xuzhihao.interceptor;

import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collection;
import java.util.Collections;
import java.util.HashSet;
import java.util.List;
import java.util.Map;
import java.util.Objects;
import java.util.Properties;
import java.util.Set;
import java.util.stream.Collectors;

import org.apache.ibatis.cache.CacheKey;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlCommandType;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.apache.ibatis.plugin.Signature;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;
import org.apache.log4j.Logger;
import org.springframework.core.convert.ConversionService;
import org.springframework.core.convert.support.DefaultConversionService;
import org.springframework.util.CollectionUtils;
import org.springframework.util.ReflectionUtils;

import com.xuzhihao.Appliaction;
import com.xuzhihao.interceptor.encrpyt.SensitiveConverter;
import com.xuzhihao.interceptor.encrpyt.SensitiveFormat;
import com.xuzhihao.interceptor.encrpyt.SensitiveType;

import cn.hutool.core.util.ObjectUtil;

/**
 * 字段加解脱敏拦截器
 * 
 * @author Administrator
 *
 */
@Intercepts({
		@Signature(type = Executor.class, method = "query", args = { MappedStatement.class, Object.class,
				RowBounds.class, ResultHandler.class }),
		@Signature(type = Executor.class, method = "query", args = { MappedStatement.class, Object.class,
				RowBounds.class, ResultHandler.class, CacheKey.class, BoundSql.class }),
		@Signature(type = Executor.class, method = "update", args = { MappedStatement.class, Object.class }) })
public class FieldEncryptInterceptor implements Interceptor {

	private static Logger log = Logger.getLogger(Appliaction.class);

	@Override
	public Object intercept(Invocation invocation) throws Throwable {
		Object[] args = invocation.getArgs();
		MappedStatement ms = (MappedStatement) args[0];// MappedStatement
		Object parameter = args[1];// 参数
		SensitiveConverter sensitiveConverter = getMethodAnnotations(ms);
		if (Objects.isNull(sensitiveConverter)) {
			return invocation.proceed();
		} else {
			SqlCommandType sqlCommandType = ms.getSqlCommandType();
			if (SqlCommandType.UPDATE.equals(sqlCommandType)
					&& SensitiveType.UPDATE.equals(sensitiveConverter.value())) {
				updateParameters(sensitiveConverter, ms, parameter);
				log.info("数据入库敏感词过滤");
			}
			Object object = invocation.proceed();
			if (SqlCommandType.SELECT.equals(sqlCommandType)
					&& SensitiveType.SELECT.equals(sensitiveConverter.value())) {
				log.info("数据显示脱敏");
				decrypt(object);
			}
			return object;
		}
	}

	/**
	 * 对象脱敏
	 * 
	 * @param object
	 * @return
	 */
	private Object decrypt(Object object) {
		if (Objects.nonNull(object)) {
			if (object instanceof List) {
				encryptByAnnByList((List<Object>) object);
			} else {
				encryptByAnnByList(Collections.singletonList(object));
			}
		}
		return object;
	}

	/**
	 * 修改入参信息
	 * 
	 * @param sensitiveConverter
	 * @param ms
	 * @param parameter
	 */
	private void updateParameters(SensitiveConverter sensitiveConverter, MappedStatement ms, Object parameter) {
		if (!(parameter instanceof Map)) {
			encryptByAnnByList(Collections.singletonList(parameter));
		} else {
			Map<String, Object> map = getParameterMap(parameter);
			map.forEach((k, v) -> {
				if (v instanceof Collection) {
					encryptByAnnByList((List<Object>) v);
				} else {
					encryptByAnnByList(Collections.singletonList(v));
				}
			});
		}
	}

	/**
	 * 获取参数的map 集合
	 * 
	 * @param parameter
	 * @return
	 */
	private Map<String, Object> getParameterMap(Object parameter) {
		Set<Integer> hashCodeSet = new HashSet<>();
		return ((Map<String, Object>) parameter).entrySet().stream().filter(e -> Objects.nonNull(e.getValue()))
				.filter(r -> {
					if (!hashCodeSet.contains(r.getValue().hashCode())) {
						hashCodeSet.add(r.getValue().hashCode());
						return true;
					}
					return false;
				}).collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
	}

	private void encryptByAnnByList(List<Object> list) {
		// 获取字段上的注解值
		if (CollectionUtils.isEmpty(list)) {
			return;
		}
		Set<Field> annotationFields = this.getFields(list.get(0)).stream()
				.filter(field -> field.isAnnotationPresent(SensitiveFormat.class)).collect(Collectors.toSet());

		ConversionService sharedInstance = DefaultConversionService.getSharedInstance();// 待整合到@Bean

		list.forEach(obj -> {
			getFields(obj).stream().filter(annotationFields::contains).forEach(field -> {
				Object value = this.getField(field, obj);
				String annovalue = field.getDeclaredAnnotation(SensitiveFormat.class).value();
				if (Objects.nonNull(value) & ObjectUtil.isNotEmpty(annovalue)) {
					ReflectionUtils.setField(field, obj,
							sharedInstance.convert(tuomin(value.toString(), annovalue), field.getType()));
				}
			});
		});
	}

	private String tuomin(String submsg, String key) {
		if ("realName".equals(key)) {
			return SensitiveInfoUtils.chineseName(submsg);// 姓名
		}
		if ("idCard".equals(key)) {
			return SensitiveInfoUtils.idCard(submsg);// 身份证号
		}
		if ("fixedPhone".equals(key)) {
			return SensitiveInfoUtils.fixedPhone(submsg);// 固定电话
		}
		if ("bankCard".equals(key)) {
			return SensitiveInfoUtils.bankCard(submsg);// 银行卡号
		}
		if ("mobilePhone".equals(key)) {
			return SensitiveInfoUtils.mobilePhone(submsg);// 手机号
		}
		if ("address".equals(key)) {
			return SensitiveInfoUtils.address(submsg, 6);// 地址
		}
		if ("email".equals(key)) {
			return SensitiveInfoUtils.email(submsg);// email
		}
		if ("cnapsCode".equals(key)) {
			return SensitiveInfoUtils.cnapsCode(submsg);// 公司开户银行联号
		}
		return submsg;
	}

	/**
	 * 获取字段
	 * 
	 * @since 2018-09-28 16:49:46
	 * @param
	 * @return
	 */
	private Object getField(Field field, Object obj) {
		ReflectionUtils.makeAccessible(field);
		return ReflectionUtils.getField(field, obj);
	}

	private List<Field> getFields(Object obj) {
		List<Field> fieldList = new ArrayList<>();
		Class tempClass = obj.getClass();
		// 当父类为null的时候说明到达了最上层的父类(Object类).
		while (tempClass != null) {
			fieldList.addAll(Arrays.asList(tempClass.getDeclaredFields()));
			// 得到父类,然后赋给自己
			tempClass = tempClass.getSuperclass();
		}
		return fieldList;
	}

	/**
	 * 得到方法上面设置的脱敏规则
	 * 
	 * @param ms
	 * @return
	 */
	private SensitiveConverter getMethodAnnotations(MappedStatement ms) {
		String namespace = ms.getId();
		String className = namespace.substring(0, namespace.lastIndexOf("."));
		String methodName = ms.getId().substring(ms.getId().lastIndexOf(".") + 1);
		Method[] mes;
		try {
			mes = Class.forName(className).getMethods();
			for (Method m : mes) {
				if (m.getName().equals(methodName)) {
					if (m.isAnnotationPresent(SensitiveConverter.class)) {
						return m.getAnnotation(SensitiveConverter.class);
					}
				}
			}
		} catch (ClassNotFoundException e) {
			e.printStackTrace();
		}
		return null;
	}

	@Override
	public Object plugin(Object target) {
		return Plugin.wrap(target, this);
	}

	@Override
	public void setProperties(Properties properties) {

	}

}

```

## 5. mybaits和spring整合是如何用到FactoryBean

```java
@ComponentScan("com.xuzhihao")
@MyScanner("com.xuzhihao.mapper")
public class Appliaction {

	@Bean
	public SqlSessionFactory sqlSessionFactory() throws IOException {
		String resource = "mybatis-config.xml";
		InputStream inputStream = Resources.getResourceAsStream(resource);
		SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
		return sqlSessionFactory;
	}

	public static void main(String[] args) throws IOException {
		try (AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext()) {
			applicationContext.register(Appliaction.class);// 配置类
			applicationContext.refresh();
			UserService userService = applicationContext.getBean("userService", UserService.class);
			userService.save();
		}
	}

}
```

```java
package com.xuzhihao.spring;

import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.springframework.beans.factory.FactoryBean;

public class MyBatisFactoryBean implements FactoryBean<Object> {
	private Class<?> mapperClass;

	public MyBatisFactoryBean(Class<?> mapperClass) {
		this.mapperClass = mapperClass;
	}

	private SqlSession sqlSession;
	
	@Override
	public Object getObject() throws Exception {
		Object mapper =sqlSession.getMapper(mapperClass);
		return mapper;
		
	}

	@Override
	public Class<?> getObjectType() {
		// TODO Auto-generated method stub
		return mapperClass;
	}

	public SqlSession getSqlSession() {
		return sqlSession;
	}

	public void setSqlSession(SqlSessionFactory sqlSessionFactory) {
		sqlSessionFactory.getConfiguration().addMapper(mapperClass);
		this.sqlSession = sqlSessionFactory.openSession();
	}

}

```

```java
package com.xuzhihao.spring;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

import org.springframework.beans.factory.FactoryBean;

public class SimpleFactoryBean implements FactoryBean<Object> {
	private Class<?> mapperClass;

	public SimpleFactoryBean(Class<?> mapperClass) {
		this.mapperClass = mapperClass;
	}

	@Override
	public Object getObject() throws Exception {
		Object proxyInstance = Proxy.newProxyInstance(SimpleFactoryBean.class.getClassLoader(),
				new Class[] { mapperClass }, new InvocationHandler() {
					@Override
					public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
						return null;
					}
				});
		return proxyInstance;
	}

	@Override
	public Class<?> getObjectType() {
		// TODO Auto-generated method stub
		return mapperClass;
	}

}

```

## 6. #和$的区别和genericTokenParser解析器

```java
package org.apache.ibatis.parsing;

import java.util.HashMap;
import java.util.Map;

/**
 * sql解析测试
 * 
 * @author admin
 *
 */
public class GenericTokenParserTest {
	public static void main(String[] args) {
		Map<String, Object> mapper = new HashMap<String, Object>();
		mapper.put("name", "张三");
		mapper.put("pwd", "123456");
		// 先初始化一个handler
		GenericTokenParser genericTokenParser = new GenericTokenParser("${", "}", new TokenHandler() {
			@Override
			public String handleToken(String content) {
				return (String) mapper.get(content);
			}
		});
		System.out.println("****" + genericTokenParser.parse("用户：${name}，你的密码是:${pwd}"));

	}
}
```

## 7. 加载mappers配置的四种方式，优先级

1.使用相对于类路径的资源引用 resource

```xml
<!-- 使用相对于类路径的资源引用 -->
<mappers>
  <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
  <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
  <mapper resource="org/mybatis/builder/PostMapper.xml"/>
</mappers>
```
2.使用完全限定资源定位符 url
```xml
<!-- 使用完全限定资源定位符（URL） -->
<mappers>
  <mapper url="file:///var/mappers/AuthorMapper.xml"/>
  <mapper url="file:///var/mappers/BlogMapper.xml"/>
  <mapper url="file:///var/mappers/PostMapper.xml"/>
</mappers>
```
3.使用映射器接口实现类的完全限定类名 class
```xml
<!-- 使用映射器接口实现类的完全限定类名 -->
<mappers>
  <mapper class="org.mybatis.builder.AuthorMapper"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  <mapper class="org.mybatis.builder.PostMapper"/>
</mappers>
```
4.将包内的映射器接口实现全部注册为映射器 package
```xml
<!-- 将包内的映射器接口实现全部注册为映射器 -->
<mappers>
  <package name="org.mybatis.builder"/>
</mappers>
```


源码验证

XMLConfigBuilder类下的this.mapperElement(root.evalNode("mappers"))

```java
// 加载mapper的形式有4种
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
        for (XNode child : parent.getChildren()) {
            // 4、package 将包内的映射器接口实现全部注册为映射器 <package //
            // name="com.bestcxx.stu.springmvc.mapper"/>
            if ("package".equals(child.getName())) {
                String mapperPackage = child.getStringAttribute("name");
                //注解方式此处放入
                configuration.addMappers(mapperPackage);
            } else {
                String resource = child.getStringAttribute("resource");
                String url = child.getStringAttribute("url");
                String mapperClass = child.getStringAttribute("class");
                // 1、resource 使用相对于类路径的资源引用 <mapper
                // resource="mybatis/mappings/UserModelMapper.xml"/>
                if (resource != null && url == null && mapperClass == null) {
                    ErrorContext.instance().resource(resource);
                    InputStream inputStream = Resources.getResourceAsStream(resource);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource,
                            configuration.getSqlFragments());
                    mapperParser.parse();
                    // 2、url 使用完全限定资源定位符 <mapper url="file:///var/mappings/UserModelMapper.xml"/>
                } else if (resource == null && url != null && mapperClass == null) {
                    ErrorContext.instance().resource(url);
                    InputStream inputStream = Resources.getUrlAsStream(url);
                    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url,
                            configuration.getSqlFragments());
                    mapperParser.parse();
                    // 3、class 使用映射器接口实现类的完全限定类名 <mapper
                    // class="com.bestcxx.stu.springmvc.mapper.UserModelMapper"/>
                } else if (resource == null && url == null && mapperClass != null) {
                    Class<?> mapperInterface = Resources.classForName(mapperClass);
                    configuration.addMapper(mapperInterface);
                } else {
                    throw new BuilderException(
                            "A mapper element may only specify a url, resource or class, but not more than one.");
                }
            }
        }
    }
}
```