# Fleet-Stability-Data-Extraction-SQL
# Purpose: 
Data extraction component that sources engine maintenance data from Redshift and prepares Atlas Data Lakes datasets for    downstream transformation pipelines in Dataiku for ingestion into Redshift Enterprise level historical data tables
# Data Flow:
  1. Amazon Redshift and Atlas Data Lakes
  2. SQL Query as shown below in Dataiku
  3. Dataiku transformations combining Atlas Data with Smartsheet manual sheets for complete weekly data tables
  4. Ingestion into redshift historical reporting tables via starfish application
  5. Enterprise Level Redshift Historical data tables
  6. Spotfire visualization tool
# Technologies
  - Amazon Redshift
  - SQL
  - Atlas Data Lakes
  - Dataiku
  - Starfish
  - Spotfire (visualization Tool similar to Tableau or PowerBI)

-- define all columns in the final table, c = CAS + CP union and t = Commit Table
Select
	c.part_number as PN,
	c.part_description as Keyword,
	c.source_name as source,
	c.quantity_received as Prev_Week_Deliveries,
	-- Previous week total commits for this part number
	Sum(case when t.metric_date >= current_date -7 and t.metric_date < current_date then t.fiscal_qty else 0 end) 
	over (partition by c.part_number) as Prev_Week_Commits, 
	-- Current week total commits for this part number
	Sum( case when t.metric_date >= current_date and t.metric_date < current_date + 7 then t.fiscal_qty else 0 end) 
	over (partition by c.part_number) as Current_Week_Commits,
	t.metric_date::date as commit_date,
	c.receive_date as actual_date,
	t.metric_name,
	c.destination_org,
	c.source_system
-- Union CAS and CP together to show all actuals data and call the new table c
From (
	-- CAS
	select part_number, receive_date, quantity_received, source_system, source_name, part_number as part_description, source_code as destination_org
	from ia_eng.sc_logistics_cas_t
	where receive_date is not null
	and receive_date >= current_date - 7
	and destination_org in ('CPL', 'CSO')
	
	union all

	-- CP
	select part_number, received_date::date as receive_date, quantity_received, source_system, source_name, part_description, destination_org
	from ia_eng.palantir_sc_logistics_cp 
	where receive_date is not null 
	and receive_date >= current_date - 7
	and destination_org in ('CPL', 'CSO')) c

--Join c and t together to final table adding commit data
left join ia_eng.s3_palantir_commits t
	on c.part_number = t.part_number
	and t.order_plant_code in ('CPL', 'CSO')
	and t.metric_name = 'COMMIT' 
	and t.metric_date >= current_date - 7 
	and t.metric_date <= current_date + 7
	
where c.receive_date >= current_date - 7
	and c.receive_date <= current_date

-- shows the results from newest to oldest relative to the current date
order by c.receive_date desc;
