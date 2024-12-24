select * from netflix_raw
create TABLE [dbo].[netflix_raw](
	[show_id] [varchar](10) NULL,
	[type] [varchar](10) NULL,
	[title] [nvarchar](200) NULL,
	[director] [varchar](250) NULL,
	[cast] [varchar](1000) NULL,
	[country] [varchar](150) NULL,
	[date_added] [varchar](120) NULL,
	[release_year] [int] NULL,
	[rating] [varchar](10) NULL,
	[duration] [varchar](10) NULL,
	[listed_in] [varchar](100) NULL,
	[description] [varchar](500) NULL
) 

--Remove duplicates
select show_id ,count(1)
from netflix_raw
group by show_id
having count(1) >1 

select * from netflix_raw
where concat (upper (title ),type) in (
select concat(upper (title) ,type)
from netflix_raw
group by title,type
having count (*) > 1)
order by title

with cte as (select * ,ROW_NUMBER () over (partition by title,type order by show_id ) as rn from netflix_raw  )
select show_id,type,title,director,cast,country,cast(date_added as date) as dateadded, rating,
case when duration is null then rating else duration end as duration  into netflix from cte
where rn = 1;

select show_id,trim(value) as country
into netflix_country
from netflix_raw
cross apply string_split(country,',')


select show_id,trim(value) as director
into netflix_director
from netflix_raw
cross apply string_split(director,',')

select show_id,trim(value) as cast
into netflix_cast
from netflix_raw
cross apply string_split(cast,',')

select show_id,trim(value) as genre
into netflix_genre
from netflix_raw
cross apply string_split(listed_in,',')

--populate missing values in country,duration,columns
insert into netflix_country
select show_id,m.country
from netflix_raw nr 
inner join (select director ,country from netflix_country
nc inner join netflix_director nd on nc.show_id= nd.show_id
group by director,country)m on nr.director =m.director
where nr.country is null

select director ,country from netflix_country
nc inner join netflix_director nd on nc.show_id= nd.show_id
group by director,country

SELECT nc.show_id, nc.country, nd.director
FROM netflix_country nc
INNER JOIN netflix_director nd ON nc.show_id = nd.show_id
order by show_id;

/* 1. for each director count the no of movies and tv shows created by them in separate columns
for directors who have creates tv shows and movies both*/

select nd.director,
count(distinct case when nc.type = 'movie' then nc.show_id end) as no_of_movies,
count(distinct case when nc.type = 'tv show' then nc.show_id end) as no_of_Tv_show from netflix nc
inner join netflix_director nd on nc.show_id=nd.show_id
group by nd.director
having count(distinct nc.type)>1

-- which country has highest number of comedy movies

select  top 1 nc.country ,count(distinct ng.show_id) as no_of_movies
from netflix_genre ng
inner join netflix_country nc on ng.show_id = nc.show_id
inner join netflix n on ng.show_id = nc.show_id
where ng.genre = 'comedies'
and n.type = 'movie'
group by nc.country
order by no_of_movies desc



-- 3.for each year (as per date added to netflix),which director has maximun number of movies released
with cte as 
        (select year(dateadded) as year_added ,nd.director ,count(nt.show_id) as no_of_movies from netflix nt
        inner join netflix_director nd on nt.show_id=nd.show_id
        where type = 'movie'
group by year(dateadded),nd.director),
with cte2 as
         (select * , ROW_NUMBER() over (partition by year_added ,director)
          from cte)
select * from cte2 where rn = 1

--4. what is average duration of movies in each genre;
select ng.genre,avg(cast( replace(duration ,' min','') as int)) as average_duration from netflix nt
inner join netflix_genre ng on nt.show_id = ng.show_id
where type = 'movie'
group by ng.genre


--5. Find the list of directors who have created horror and comedy movies both
--display director names along with number of comedy and horror movies directed by them

select nd.director,count(distinct case when ng.genre = 'comedies' then nt.show_id end ) as no_of_comedy,
count(distinct case when ng.genre = 'horror movies' then nt.show_id end ) as no_of_horror from netflix nt
inner join netflix_director nd on nt.show_id=nd.show_id
inner join netflix_genre ng on nt.show_id = ng.show_id
where nt.type = 'movie' and ng.genre in ('comedies','horror movies')
group by nd.director
having COUNT(ng.genre)=2
