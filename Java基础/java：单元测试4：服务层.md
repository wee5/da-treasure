# java：单元测试4：服务层

3种方法：

1.用真实数据库测试增删查改,dao层也一起测试了。因为数据库内容随时在变,每次测试前先针对每个测试方法跑一下sql脚本查出结果,再修改单测方法的Assert,然后跑单测。如果有50个单测,就要跑50遍不同sql,修改50个单测方法。如果需要数据,先插入一些数据。（这是我们组长要求的做法。）

2.测试前清空相关表的内容,再插入数据,根据插入的数据进行Assert。这样将dao层也测试到了,不需要每次测试前先跑sql。

3.用mock测试,隔离dao层,根据模拟出来的数据进行Assert判断。这里尽量保证dao层的正确性,如果需要dao层测试,再考虑。

第一种方法,操作繁琐,浪费时间。

第二种方法,在插入数据时需要保证外键的正确性,比第一种方法要好。

第三种,未测试dao层,比较快