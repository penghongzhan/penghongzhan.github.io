# connection&session&query

datasource -> sessionFactory(mybatis) -> sqlSessionTemplate -> session -> query

session涉及到事务，还会有PlatformTransactionManager，spring用来进行事务管理的，事务一定是在session内部的