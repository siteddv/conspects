```sql
-- create staging layer

drop table if exists staging.last_update;
create table staging.last_update (
	table_name varchar(50) not null,
	update_dt timestamp not null
);

drop table if exists staging.film;
CREATE TABLE staging.film (
	film_id int NOT NULL,
	title varchar(255) NOT NULL,
	description text NULL,
	release_year year NULL,
	language_id int2 NOT NULL,
	rental_duration int2 NOT NULL,
	rental_rate numeric(4, 2) NOT NULL,
	length int2 NULL,
	replacement_cost numeric(5, 2) NOT NULL,
	rating mpaa_rating NULL,
	last_update timestamp NOT NULL,
	special_features _text NULL,
	fulltext tsvector NOT NULL
);

drop table if exists staging.inventory;
CREATE TABLE staging.inventory (
	inventory_id int NOT NULL,
	film_id int2 NOT NULL,
	store_id int2 NOT NULL,
	last_update timestamp NOT NULL,
	deleted timestamp NULL
);

drop table if exists staging.rental;
CREATE TABLE staging.rental (
	rental_id int NOT NULL,
	rental_date timestamp NOT NULL,
	inventory_id int4 NOT NULL,
	customer_id int2 NOT NULL,
	return_date timestamp NULL,
	staff_id int2 NOT NULL,
	last_update timestamp NOT NULL,
	deleted timestamp NULL
);

-- create  staging procedures

create or replace function staging.get_last_update_table(table_name varchar) returns timestamp
as $$
	begin
		return coalesce( 
			(
				select
					max(update_dt)
				from
					staging.last_update lu
				where 
					lu.table_name = get_last_update_table.table_name
			),
			'1900-01-01'::date	
		);
	end;
$$ language plpgsql;

create or replace procedure staging.set_table_load_time(table_name varchar, current_update_dt timestamp default now())
as $$
	begin
		INSERT INTO staging.last_update
		(
			table_name, 
			update_dt
		)
		VALUES(
			table_name, 
			current_update_dt
		);
	end;
$$ language plpgsql;


create or replace procedure staging.film_load() 
as $$
	begin 
		delete from staging.film;
		insert
			into
			staging.film
		(
			film_id,
			title,
			description,
			release_year,
			language_id,
			rental_duration,
			rental_rate,
			length,
			replacement_cost,
			rating,
			last_update,
			special_features,
			fulltext
		)
		select
			film_id,
			title,
			description,
			release_year,
			language_id,
			rental_duration,
			rental_rate,
			length,
			replacement_cost,
			rating,
			last_update,
			special_features,
			fulltext
		from
			film_src.film;
	end;
$$ language plpgsql;

create or replace procedure staging.inventory_load()
as $$
	begin 
		delete from staging.inventory;
	
		insert
			into
			staging.inventory
		(
			inventory_id,
			film_id,
			store_id,
			last_update,
			deleted
		)
		select
			inventory_id,
			film_id,
			store_id,
			last_update,
			deleted
		from
			film_src.inventory;
	end;
$$ language plpgsql;

create or replace procedure staging.rental_load(current_update_dt timestamp)
as $$
	declare 
		last_update_dt timestamp;
	begin
		last_update_dt = staging.get_last_update_table('staging.rental');
	
		delete from staging.rental;

		insert into staging.rental
		(
			rental_id,
			rental_date,
			inventory_id,
			customer_id,
			return_date,
			staff_id,
			last_update,
			deleted
		)
		select 
			rental_id, 
			rental_date, 
			inventory_id, 
			customer_id, 
			return_date, 
			staff_id,
			last_update,
			deleted
		from
			film_src.rental
		where 
			deleted >= last_update_dt
			or last_update >= last_update_dt;
		
		call staging.set_table_load_time('staging.rental', current_update_dt);
	end;

$$ language plpgsql;

-- создание таблиц ODS слоя

drop table if exists ods.film;
CREATE TABLE ods.film (
	film_id int NOT NULL,
	title varchar(255) NOT NULL,
	description text NULL,
	release_year year NULL,
	language_id int2 NOT NULL,
	rental_duration int2 NOT NULL,
	rental_rate numeric(4, 2) NOT NULL,
	length int2 NULL,
	replacement_cost numeric(5, 2) NOT NULL,
	rating mpaa_rating NULL,
	last_update timestamp NOT NULL,
	special_features _text NULL,
	fulltext tsvector NOT NULL
);

drop table if exists ods.inventory;
CREATE TABLE ods.inventory (
	inventory_id int NOT NULL,
	film_id int2 NOT NULL,
	store_id int2 NOT NULL,
	last_update timestamp NOT NULL,
	deleted timestamp NULL
);

drop table if exists ods.rental;
CREATE TABLE ods.rental (
	rental_id int NOT NULL,
	rental_date timestamp NOT NULL,
	inventory_id int4 NOT NULL,
	customer_id int2 NOT NULL,
	return_date timestamp NULL,
	staff_id int2 NOT NULL,
	last_update timestamp NOT NULL,
	deleted timestamp NULL
);

-- create ods procedures

create or replace procedure ods.film_load() 
as $$
	begin 
		delete from ods.film;
		insert
			into
			ods.film
		(
			film_id,
			title,
			description,
			release_year,
			language_id,
			rental_duration,
			rental_rate,
			length,
			replacement_cost,
			rating,
			last_update,
			special_features,
			fulltext
		)
		select
			film_id,
			title,
			description,
			release_year,
			language_id,
			rental_duration,
			rental_rate,
			length,
			replacement_cost,
			rating,
			last_update,
			special_features,
			fulltext
		from
			staging.film;
	end;
$$ language plpgsql;

create or replace procedure ods.inventory_load()
as $$
	begin 
		delete from ods.inventory;
	
		insert
			into
			ods.inventory
		(
			inventory_id,
			film_id,
			store_id,
			last_update,
			deleted
		)
		select
			inventory_id,
			film_id,
			store_id,
			last_update,
			deleted
		from
			staging.inventory;
	end;
$$ language plpgsql;

create or replace procedure ods.rental_load()
as $$
	begin	
		delete from ods.rental r
		where r.rental_id in (
			select
				sr.rental_id 
			from staging.rental sr
		);

		insert into ods.rental
		(
			rental_id,
			rental_date,
			inventory_id,
			customer_id,
			return_date,
			staff_id,
			last_update,
			deleted
		)
		select 
			rental_id, 
			rental_date, 
			inventory_id, 
			customer_id, 
			return_date, 
			staff_id,
			last_update,
			deleted
		from
			staging.rental;
	end;

$$ language plpgsql;

-- create tables ref layer

drop table if exists ref.film;
create table ref.film (
	film_sk serial NOT null,
	film_nk int NOT null
);

drop table if exists ref.inventory;
create table ref.inventory (
	inventory_sk serial NOT null,
	inventory_nk int NOT null
);

drop table if exists ref.rental;
create table ref.rental (
	rental_sk serial NOT null,
	rental_nk int NOT null
);

-- create ref procedures

create or replace procedure film_id_sync()
as $$
	begin 
		-- находим все записи, которым не выдан суррогатный id
		-- выдаем им суррогатный id
		insert into ref.film (
			film_nk
		)
		select
			odf.film_id
		from 
			ods.film odf
			left join ref.film rf
				on odf.film_id = rf.film_nk
		where 
			rf.film_nk is null
		order by odf.film_id;
	end;
$$ language plpgsql;

create or replace procedure inventory_id_sync()
as $$
	begin 
		-- находим все записи, которым не выдан суррогатный id
		-- выдаем им суррогатный id
		insert into ref.inventory (
			inventory_nk
		)
		select
			odi.inventory_id
		from 
			ods.inventory odi
			left join ref.inventory ri
				on odi.inventory_id = ri.inventory_nk
		where 
			ri.inventory_nk is null
		order by odi.inventory_id;
	end;
$$ language plpgsql;

create or replace procedure rental_id_sync()
as $$
	begin 
		-- находим все записи, которым не выдан суррогатный id
		-- выдаем им суррогатный id
		insert into ref.rental (
			rental_nk
		)
		select
			odr.rental_id
		from 
			ods.rental odr
			left join ref.rental rr
				on odr.rental_id = rr.rental_nk
		where 
			rr.rental_nk is null
		order by odr.rental_id;
	end;
$$ language plpgsql;

-- full load

create or replace procedure full_load()
as $$
	declare 
 		current_update_dt timestamp = now();
	begin
		call staging.film_load();
		call staging.inventory_load();
		call staging.rental_load(current_update_dt);
	
		call ods.film_load();
		call ods.inventory_load();
		call ods.rental_load();
	
		call film_id_sync();
		call inventory_id_sync();
		call rental_id_sync();
	end;
$$ language plpgsql;

call full_load();
```