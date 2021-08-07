# gamedev-project2

with user_address as --физический адрес
(select a.user_id,
case
when a.country = 'Russia' then 'yes'
when a.country is null then null
else 'no'
end as is_russia
from gd2.addresses a),

user_phone as --номер телефона
(select u.id as user_id,
case
when left(u.phone, 1)='7' and length(u.phone) = 11 then 'yes'
when u.phone is null then null
else 'no'
end as is_russia
from gd2.users1 u),

users_with_geo as
(select u.id as user_id, 
coalesce(ua.is_russia, up.is_russia) as is_russia
from gd2.users1 u
left join user_address ua on u.id = ua.user_id
left join user_phone up on u.id = up.user_id
where coalesce(ua.is_russia, up.is_russia) is not null
order by 2),

reg_purchases as
(select uwg.user_id, date_trunc('month', p.created_at) month_purchase, 
count(distinct case when uwg.is_russia='yes' then p.id end) russia_purchase, 
count(distinct case when uwg.is_russia='no' then p.id end ) other_purchase
from users_with_geo uwg 
join
gd2.purchases p on uwg.user_id=p.user_id and p.state='successful'
group by 1,2),

first_purchases as
(select distinct p.user_id,
 min(date_trunc('month', p.created_at)) min_date
from gd2.purchases p
where p.state='successful'
group by 1
order by 2),

cohorts_russia as
(select fp.min_date::date, rp.month_purchase::date,
count(distinct fp.user_id) cnt_id
from first_purchases fp 
left join users_with_geo uwg on fp.user_id=uwg.user_id
join reg_purchases rp on rp.user_id=fp.user_id
where uwg.is_russia='yes' --для не России поставить no
group by 1,2
order by 1,2),

cohorts_not_russia as
(select fp.min_date::date, rp.month_purchase::date,
count(distinct fp.user_id) as cnt_id
from first_purchases fp 
left join users_with_geo uwg on fp.user_id=uwg.user_id
join reg_purchases rp on rp.user_id=fp.user_id
where uwg.is_russia='no' --для не России поставить no
group by 1,2
order by 1,2),

retention_cohorts as 
(select min_date, month_purchase, cnt_id,
cnt_id *100  / first_value(cnt_id) over (partition by min_date
							order by min_date) as retention
from cohorts_not_russia),

gaps as 
(select p.user_id,
lead(p.created_at) over (partition by p.user_id order by p.created_at) - p.created_at as diff
from gd2.purchases p
where state = 'successful'),

avg_bill as
(select p.user_id,
avg(p.amount) avg_amount
from gd2.purchases p
where p.state='successful'
group by 1),

core_players_june as
(select
distinct g.user_id unique_users, sum(amount) as sum_core_june
from gaps g join avg_bill ab on g.user_id=ab.user_id
join gd2.purchases p on ab.user_id=p.user_id
where ab.avg_amount>750
and p.created_at between '2020-05-01' and '2020-06-30' and state = 'successful'
group by 1 
having avg(g.diff)<=28),

share as (select sum(sum_core_june) *100 / 
(select sum (amount)
from gd2.purchases
where state = 'successful')
from core_players_june),


revenues_by_segments as 
(select p.user_id, sum(p.amount),
case when p.user_id = cpj.unique_users then 'core' else 'not_core' end core_player
from gd2.purchases p
left join core_players_june cpj on p.user_id = cpj.unique_users
where created_at between '2020-06-01' and '2020-06-30'
group by 1,3)


select
round(sum
(case when p.user_id in
(select unique_users
from core_players_june cpj)
 then p.amount
end) / sum(p.amount)::numeric*100) as share       
from gd2.purchases p 
where state = 'successful'
