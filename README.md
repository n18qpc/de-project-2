--1. Создать справочник стоимости доставки в страны 
-- Имя таблицы - shipping_country_rates
-- данные из shipping, поля shipping_country, shipping_country_base_rate
CREATE TABLE public.shipping_country_rates (
	id 						   SERIAL,
	shipping_country 		   TEXT,
	shipping_country_base_rate NUMERIC(14,3),
	PRIMARY KEY (id)
);

-- Заполним таблицу shipping_country_rates данными из таблицы shipping
INSERT INTO public.shipping_country_rates (shipping_country, shipping_country_base_rate) 
SELECT 
	s.shipping_country,
	s.shipping_country_base_rate 
FROM public.shipping s 
GROUP BY s.shipping_country, s.shipping_country_base_rate
ORDER BY s.shipping_country, s.shipping_country_base_rate;

-- Проверим, что данные загружены
SELECT * FROM public.shipping_country_rates;


--2. Создать справочник тарифов доставки вендора по договору
-- Имя таблицы - shipping_agreement 
-- данные из shipping, поле vendor_agreement_description 
CREATE TABLE public.shipping_agreement (
	agreementid 	     BIGINT,
	agreement_number 	 TEXT,
	agreement_rate 		 NUMERIC(14,3),
	agreement_commission NUMERIC(14,3),
	PRIMARY KEY (agreementid)
);

-- Заполним таблицу shipping_agreement данными из таблицы shipping
INSERT INTO public.shipping_agreement (agreementid, agreement_number, agreement_rate, agreement_commission)
WITH cte AS (
SELECT
	--vendor_agreement_description,
	regexp_split_to_array(vendor_agreement_description, ':+') AS vendor_agreement_array
FROM public.shipping
)
SELECT
	CAST(vendor_agreement_array[1] AS BIGINT) AS agreementid,
	CAST(vendor_agreement_array[2] AS TEXT) AS agreement_number,
	CAST(vendor_agreement_array[3] AS NUMERIC(14,3)) AS agreement_rate,
	CAST(vendor_agreement_array[4] AS NUMERIC(14,3)) AS agreement_commission
FROM cte
GROUP BY 1,2,3,4;

-- Проверим, что данные загружены
SELECT * FROM public.shipping_agreement ORDER BY agreementid;


--3. Создать справочник о типах доставки
-- Имя таблицы - shipping_transfer 
-- данные из shipping, поле shipping_transfer_description, shipping_transfer_rate
CREATE TABLE public.shipping_transfer (
	id 					   SERIAL,
	transfer_type 		   TEXT,
	transfer_model 		   TEXT,
	shipping_transfer_rate NUMERIC(14,3),
	PRIMARY KEY (id)
);

-- Заполним таблицу shipping_transfer данными из таблицы shipping
INSERT INTO public.shipping_transfer (transfer_type, transfer_model, shipping_transfer_rate)
WITH cte AS (
SELECT 
	regexp_split_to_array(shipping_transfer_description, ':+') AS shipping_transfer_array,
	shipping_transfer_rate
FROM public.shipping
)
SELECT
	CAST(shipping_transfer_array[1] AS TEXT) AS transfer_type,
	CAST(shipping_transfer_array[2] AS TEXT) AS transfer_model,
	CAST(shipping_transfer_rate AS NUMERIC(14,3)) AS shipping_transfer_rate
FROM cte
GROUP BY 1,2,3;

-- Проверим, что данные загружены
SELECT * FROM public.shipping_transfer ORDER BY id;


--4. Создать таблицу с уникальными доставками
-- Имя таблицы shipping_info 
CREATE TABLE public.shipping_info (
	shippingid               BIGINT,
	vendorid                 BIGINT,
	payment_amount 		     NUMERIC(14,3),
	shipping_plan_datetime   TIMESTAMP,
	shipping_transfer_id     BIGINT,
	shipping_agremeent_id    BIGINT,
	shipping_country_rate_id BIGINT,
	PRIMARY KEY (shippingid),
	FOREIGN KEY (shipping_transfer_id) REFERENCES shipping_transfer (id) ON UPDATE CASCADE,
	FOREIGN KEY (shipping_agremeent_id) REFERENCES shipping_agreement (agreementid) ON UPDATE CASCADE,
	FOREIGN KEY (shipping_country_rate_id) REFERENCES shipping_country_rates (id) ON UPDATE CASCADE
);

-- Заполним таблицу shipping_info данными из таблицы shipping и связанными таблицами
INSERT INTO public.shipping_info (shippingid, vendorid, payment_amount, shipping_plan_datetime, shipping_transfer_id, shipping_agremeent_id, shipping_country_rate_id)
SELECT DISTINCT
	s.shippingid, 
	s.vendorid,
	s.payment_amount,
	s.shipping_plan_datetime,
	st.id AS shipping_transfer_id,
	CAST((regexp_split_to_array(s.vendor_agreement_description, ':+'))[1] AS BIGINT) AS shipping_agremeent_id,
	scr.id AS shipping_country_rate_id
FROM public.shipping s 
LEFT JOIN public.shipping_transfer st
	ON (regexp_split_to_array(s.shipping_transfer_description, ':+'))[1]  = st.transfer_type 
    AND (regexp_split_to_array(s.shipping_transfer_description, ':+'))[2] = st.transfer_model
LEFT JOIN public.shipping_country_rates scr 
	ON s.shipping_country = scr.shipping_country;

-- Проверим, что данные загружены
SELECT * FROM public.shipping_info ORDER BY shippingid;


--5. Создать таблицу статусов о доставке 
-- Имя таблицы - shipping_status
CREATE TABLE public.shipping_status (
	shippingid                   BIGINT,
	status                       TEXT,
	state                        TEXT,
	shipping_start_fact_datetime TIMESTAMP,
	shipping_end_fact_datetime   TIMESTAMP
);

-- Заполним таблицу shipping_status данными из таблицы shipping
INSERT INTO public.shipping_status (shippingid, status, state, shipping_start_fact_datetime, shipping_end_fact_datetime)
WITH cte AS (
	SELECT 
		shippingid,
		MAX(CASE WHEN state = 'booked' THEN state_datetime ELSE NULL END) AS shipping_start_datetime,
		MAX(CASE WHEN state = 'recieved' THEN state_datetime ELSE NULL END) AS shipping_end_datetime,
		MAX(state_datetime) AS max_state_datetime
	FROM public.shipping 
	GROUP BY shippingid
)
SELECT 
	cte.shippingid,
	s.status,
	s.state,
	cte.shipping_start_datetime AS shipping_start_fact_datetime,
	cte.shipping_end_datetime AS shipping_end_fact_datetime
FROM cte
LEFT JOIN public.shipping s 
	ON cte.shippingid = s.shippingid 
	AND cte.max_state_datetime = s.state_datetime 
ORDER BY shippingid;

-- Проверим, что данные загружены
SELECT * FROM public.shipping_status ORDER BY shippingid;


--6. Создать представление shipping_datamart 
-- Имя view - shipping_datamart
CREATE OR REPLACE VIEW shipping_datamartAS 
SELECT 
	si.shippingid,
	si.vendorid,
	st.transfer_model,
	DATE_PART('day', AGE(shipping_end_fact_datetime, shipping_start_fact_datetime)) AS full_day_at_shipping,
	CASE WHEN ss.shipping_end_fact_datetime > si.shipping_plan_datetime 
		 	THEN 1 
		 	ELSE 0 
		 END AS is_delay,
	CASE WHEN ss.status = 'finished' 
	     	THEN 1 
	     	ELSE 0 
	     END AS is_shipping_finish,
	CASE WHEN ss.shipping_end_fact_datetime > si.shipping_plan_datetime 
	     	THEN DATE_PART('day', AGE(ss.shipping_end_fact_datetime, si.shipping_plan_datetime)) 
	     	ELSE 0 
	     END AS delay_day_at_shipping,
	si.payment_amount,
	si.payment_amount * (scr.shipping_country_base_rate + sa.agreement_rate + st.shipping_transfer_rate) AS vat,
	si.payment_amount * sa.agreement_commission AS profit
FROM public.shipping_info si
LEFT JOIN public.shipping_transfer st
	ON si.shipping_transfer_id = st.id 
LEFT JOIN public.shipping_status ss 
	ON si.shippingid = ss.shippingid
LEFT JOIN public.shipping_country_rates scr 
	ON si.shipping_country_rate_id = scr.id 
LEFT JOIN public.shipping_agreement sa 
	ON si.shipping_agremeent_id = sa.agreementid;

































