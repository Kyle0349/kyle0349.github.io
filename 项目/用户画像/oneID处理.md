

# oneID概述

> 通过关联所有采集到的信息，识别出一个真实的自然人。



1、只要记录中任意一个信息相同，则认为这些记录在现实中是一个人。

| 字段       | 注释                                      |
| ---------- | ----------------------------------------- |
| oneid      | 随机唯一编号                              |
| uid        | 注册id                                    |
| device_id  | 设备id                                    |
| product_id | 产品id                                    |
| email      | 邮箱                                      |
| phone      | 电话                                      |
| exist_mark | md5(uid,device_id,product_id,email,phone) |

前面已经处理了一张用户id关系表处理 dwd.dwd_ups_one_id_total_da , 这是一张日全量表，每天将不存在旧分区的新增的用户记录（所有字段md5值不一致则认为是新增）union到这个表中，所以这个表中会存在大量可能存在有关联信息的记录，需要做id mapping，形成一张最后输出的oneID表dwd.dwd_ups_one_id_mapped_da

1. 计算出每个id值对应的用户数

   > 分别计算出相同的uid对应几个oneid，相同的device_id对应几个oneid
   >
   > 这里要注意对id字段做一次前缀加盐处理，因为有的信息字段大多是null，会导致数据倾斜

```sql
insert overwrite table dwd.dwd_ups_one_id_total_freq_da
select 
	*
	  count(case when substring(uid, -5, 5)= "null" or uid  is null then null esle oneid  end) over(partition by uid) as freq_uid 
	, count(case when substring(product_user_id, -5, 5)= "null" or product_user_id is null then null esle oneid end) over(partition by product_user_id) as freq_product_user_id 
	, count(case when substring(email, -5, 5)= "null" or email  is null then null esle oneid  end) over(partition by email) as freq_email 
	, count(case when substring(phone, -5, 5)= "null" or phone  is null then null esle oneid  end) over(partition by phone) as freq_phone 
(select
	  oneid
	, case when (uid = "" or uid  is null then concat(floor(rand() * 100, "-", "null")) else uid  end as uid 
	, case when (product_user_id = "" or product_user_id is null then concat(floor(rand() * 100, "-", "null")) else product_user_id end as product_user_id
	, case when (email = "" or email  is null then concat(floor(rand() * 100, "-", "null")) else email  end as email 
	, case when (phone = "" or phone  is null then concat(floor(rand() * 100, "-", "null")) else phone  end as phone 
	, exist_mark
from dwd.dwd_ups_one_id_total_da
where day  = "{process_day}")
```

2. 筛选出需要做id mapping 和不需要做id mapping的记录

   > 根据freq的数值判断

```sql
insert overwrite table dwd.dwd_ups_one_id_need_mapping_da
select
	  oneid
	, case when substring(uid, -5, 5) = "null" or uid is null then null esle uid end as uid
	, case when substring(product_user_id, -5, 5) = "null" or product_user_id is null then null esle product_user_id end as product_user_id
	, case when substring(email, -5, 5) = "null" or email is null then null esle email end as email
	, case when substring(phone, -5, 5) = "null" or phone is null then null esle phone end as phone
	, exist_mark
from dwd.dwd_ups_one_id_total_freq_da
where day = "${process_day}"
and ((freq_uid > 1) or (freq_product_id > 1) or (freq_email > 1) or (freq_phone > 1))
;


insert overwrite table dwd.dwd_ups_one_id_no_need_mapping_da
select
	  oneid
	, case when substring(uid, -5, 5) = "null" or uid is null then null esle uid end as uid
	, case when substring(product_user_id, -5, 5) = "null" or product_user_id is null then null esle product_user_id end as product_user_id
	, case when substring(email, -5, 5) = "null" or email is null then null esle email end as email
	, case when substring(phone, -5, 5) = "null" or phone is null then null esle phone end as phone
	, exist_mark
from dwd.dwd_ups_one_id_total_freq_da
where day = "${process_day}"
and (freq_uid <= 1 or freq_uid is null)
and (freq_product_id <= 1 or freq_product_id is null)
and (freq_email <= 1 or freq_email is null)
and (freq_phone <= 1 or freq_phone is null)


```

3. 对每个id根据业务需求先后做mapping

   > 以 product_user_id 作为关联信息做mapping

```sql
select
	  oneid_max as oneid
	, case when uid is null then uid_max else uid end as uid
	, product_user_id
	, case when email is null then email_max else email end as email
	, case when phone is null then phone_max else phone end as phone
from	
(select
  *
, max(oneid) over(partition by product_user_id) as oneid_max
, max(uid) over(partition by product_user_id) as uid_max
, max(email) over(partition by product_user_id) as email_max
, max(phone) over(partition by product_user_id) as phone_max
from dwd.dwd_ups_one_id_need_mapping_da where product_user_id is not null
union all
select
  *
, oneid as oneid_max
, uid as uid_max
, email as email_max
, phone as phone_max
from dwd.dwd_ups_one_id_need_mapping_da where product_user_id is null)
```







reference :

https://blog.csdn.net/weixin_43194923/article/details/107832666

