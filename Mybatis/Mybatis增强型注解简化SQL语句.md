// 简洁的数据库操作

@Select("SELECT * FROM users WHERE id IN (#{userIdList})")
@Lang(SimpleSelectInLangDriver.class)
List<User> selectUsersByUserId(List<Integer> userIdList);


@Insert("INSERT INTO users (#{user})")
@Lang(SimpleInsertLangDriver.class)
void insertUserDAO(User user);


@Update("UPDATE users (#{user}) WHERE id = #{id}")
@Lang(SimpleUpdateLangDriver.class)
void updateUsersById(User user);

@Select("SELECT id,user_name,password,phone,address,email FROM users (#{user})")
@Lang(SimpleSelectLangDriver.class)
void selectUserDAO(User user);