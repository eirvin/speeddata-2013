# Methodology: Processing MS2 Data into Period Speeds
## Contents
* [The Raw Data](#the-raw-data)
* [Speed by 15-minute Epoch](#speed-by-15-minute-epoch)
* [Speeds by Modeling Time Period](#speeds-by-modeling-time-period)
* [Maximum Free Flow Speed](#maximum-free-flow-speed)
* [Planning Time Index](#planning-time-index)

## The Raw Data

The table `spdidx` contains the raw data, which includes link id, epoch, day of week, day of month, year, average speed, maximum speed, minimum speed, information on whether the record is an estimate or an actual observation, number of samples, and 5th through 95th percentile speeds at 5% intervals. Note that this data is not the individual vehicle-level speed records, but aggregation of those records in 15 minute increments. To avoid individual measurement errors skewing the overall estimates, we decided to compute the average of median speed instead of average speed for each link during particular time period. Because estimates with more samples are more reliable, we weighted the median by the sample size. We also excluded estimates from our analysis and only used actual observed data. The following code multiplies median speed by sample size for all non-estimate records with a sample size greater than 10: 

``` sql
alter table spdidx
add column medianxsample numeric;

update spdidx
	set medianxsample= case
			when samples < 10 or is_estimate = true then null
			else samples*_50th
			end;
```
Because the table is quite large and the queries we will be performing will mostly involve a subset of the data (weekday travel in 2013), creating a partial index on the table will reduce processing time:

``` sql
create index spdidx_epochidx on spdidx (epoch)
where yr = 2013 and dow > 1 and dow <7;
```
It's important to run a vaccuum analyze at this point so that the database can actually use the index.

## Speed by 15-Minute Epoch ##
The `linkspd_epoch_2013` table contains one record for each combination of link and epoch, along with the weighted average speed and the average number of samples. because this table contains 96 records for each of almost 20,000 links, this table contains over a million rows and is unsuitable for being joined to shapefiles or exported for use in excel without further filtering. It is available as a csv file for other applications.

``` sql
create table linkspd_epoch_2013 as 
select link_id, epoch, sum(spdidx.medianxsample)/sum(spdidx.samples) speed, avg(spdidx.samples) sample
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0
group by link_id, epoch;

alter table linkspd_epoch_2013
add column start_time time;

update linkspd_epoch_2013
set start_time = epoch_lookup.start_time 
from epoch_lookup
where epoch_lookup.epoch = linkspd_epoch_2013.epoch
```
## Speeds by Modeling Time Period
The `linkmedspd_2013` table has a unique record for each link (TMC) and fields with average speeds at  the following nine time periods:
* **free flow:** 8:00 pm - 5:30 am
* **am shoulder 1:** 6:00 am - 7:00 am
* **am peak:** 7:00 am - 9:00 am
* **am shoulder 2:** 9:00 am - 10:00 am
* **mid-day:** 10:00 am - 2:00 pm
* **pm shoulder 1:** 2:00 pm - 4:00 pm
* **pm peak:** 4:00 pm - 6:00 pm
* **pm shoulder 2:** 6:00 pm - 8:00 pm
* **overnight:** 8:00 pm - 6:00 am

*Note that the free flow period is a subset of the overnight period, in an attempt to capture speeds when roads are at their least congested. See the [maximum free-flow](#maximum-free-flow-speed) discussion below for further elaboration on the final free flow speed calculation.*

It's good for joining to shapefiles because there is only one record per link.
``` sql
create table linkmedspd_2013 (
link_id,
ff_spd numeric,
am_shld1_spd numeric,
am_peak_spd numeric,
am_shld2_spd numeric,
midday_spd numeric,
pm_shld1_spd numeric,
pm_peak_spd numeric,
pm_shld2_spd numeric,
ovrnight_spd numeric,
am_shld1_samp bigint,
am_peak_samp bigint,
am_shld2_samp bigint,
midday_samp bigint,
pm_shld1_samp bigint,
pm_peak_samp bigint,
pm_shld2_samp bigint,
ovrnight_samp bigint,
tot_samp bigint,
max_ff_spd numeric,
max_ff_period text,
am_perc_05_median numeric,
pm_perc_05_median numeric,
am_wtd_mean_05th numeric,
pm_wtd_mean_05th numeric,
am_perc_50_median numeric,
pm_perc_50_median numeric,
am_pti numeric,
pm_pti numeric);

insert into linkmedspd_2013 (link_id)
select distinct link_id
from spdidx;

create index linkmedspd_linkidx on linkmedspd_2013(link_id);
```
again, run vaccuum analyze on the table before proceeding in order to take advantage of processing efficiencies.

The next set of queries computes the weighted average speed and sample size in the nine periods. For the am and pm peak, this script also calculates the median of the median speed, the 5th percentile of the median speed and the weighted mean of the 5th percentile speed, which may be used in planning time index calculations (see [planning time index](#planning-time-index) below).

``` sql
create or replace view linkspdff as
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) ff_spd, sum(spdidx.samples) sample
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and (spdidx.epoch >80 or spdidx.epoch < 23)
group by link_id;

alter table linkmedspd_2013
add column ff_samp;

update linkmedspd_2013
set ff_spd= linkspdff.speed,
	ff_samp= linkspdff.sample
from linkspdff
where linkspdff.link_id=linkmedspd_2013.link_id; 

Create or replace view linkspdamshld1 as 
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) am_shld1_spd, sum(spdidx.samples) sample
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and spdidx.epoch >24 and spdidx.epoch < 29
group by link_id;

update linkmedspd_2013
set am_shld1_spd= linkspdamshld1.am_shld1_spd,
	am_shld1_samp= linkspdamshld1.sample
from linkspdamshld1
where linkspdamshld1.link_id=linkmedspd_2013.link_id; 

Create or replace view linkspdampeak as 
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) am_peak_spd, sum(spdidx.samples) sample, percentile_cont(.05)within group(order by _50th) perc_05_median, sum(_05th*samples)/sum(samples) wtd_mean_05th, percentile_cont(.5)within group(order by _50th) perc_50_median
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and spdidx.epoch >28 and spdidx.epoch < 37
group by link_id;

update linkmedspd_2013
set am_peak_spd= linkspdampeak.am_peak_spd,
	am_peak_samp=linkspdampeak.am_peak_samp
from linkspdampeak
where linkspdampeak.link_id=linkmedspd_2013.link_id; 

Create or replace view linkspdamshld2 as 
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) am_shld2_spd, sum(spdidx.samples) sample
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and spdidx.epoch >36 and spdidx.epoch < 41
group by link_id;

update linkmedspd_2013
set am_shld2_spd= linkspdamshld2.am_shld2_spd,
	am_shld2_samp= linkspdamshld2.sample
from linkspdamshld2
where linkspdamshld2.link_id=linkmedspd_2013.link_id; 

Create or replace view linkspdmidday as 
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) midday_spd, sum(spdidx.samples) sample
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and spdidx.epoch >40 and spdidx.epoch < 57
group by link_id;

update linkmedspd_2013
set midday_spd= linkspdmidday.midday_spd,
	midday_samp=linkspdmidday.sample
from linkspdmidday
where linkspdmidday.link_id=linkmedspd_2013.link_id; 

Create or replace view linkspdpmshld1 as 
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) speed, sum(spdidx.samples) sample
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and spdidx.epoch >56 and spdidx.epoch < 65
group by link_id;

update linkmedspd_2013
set pm_shld1_spd= linkspdpmshld1.speed,
	pm_shld1_samp=linkspdpmshld1.sample
from linkspdpmshld1
where linkspdpmshld1.link_id=linkmedspd_2013.link_id; 

Create or replace view linkspdpmpeak as 
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) speed, sum(spdidx.samples) sample, percentile_cont(.05)within group(order by _50th) perc_05_median, sum(_05th*samples)/sum(samples) wtd_mean_05th, percentile_cont(.5)within group(order by _50th) perc_50_median
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and spdidx.epoch >64 and spdidx.epoch < 73
group by link_id;

update linkmedspd_2013
set pm_peak_spd= linkspdpmpeak.speed,
	pm_peak_samp=linkspdpmpeak.sample
from linkspdpmpeak
where linkspdpmpeak.link_id=linkmedspd_2013.link_id; 

Create or replace view linkspdpmshld2 as 
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) speed, sum(spdidx.samples) sample
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and spdidx.epoch >72 and spdidx.epoch < 81
group by link_id;

update linkmedspd_2013
set pm_shld2_spd= linkspdpmshld2.speed,
	pm_shld2_samp=linkspdpmshld2.sample
from linkspdpmshld2
where linkspdpmshld2.link_id=linkmedspd_2013.link_id; 

Create or replace view linkspdovrnight as 
select link_id, sum(spdidx.medianxsample)/sum(spdidx.samples) speed, sum(spdidx.samples) sample
from spdidx
where spdidx.yr=2013 and spdidx.dow >1 and spdidx.dow < 7 and spdidx.medianxsample > 0 and (spdidx.epoch >80 or spdidx.epoch < 25)
group by link_id;

update linkmedspd_2013
set ovrnight_spd= linkspdovrnight.speed,
	ovrnight_samp=linkspdovrnight.sample
from linkspdovrnight
where linkspdovrnight.link_id=linkmedspd_2013.link_id; 
```
The period sample fields and the total sample field represent the *total* number of observations included in the estimate. *Note: this is not the average number of samples in the period.*
``` sql
update linkmedspd_2013
set tot_samp= am_shld1_samp+am_peak_samp+am_shld2_samp+midday_samp+pm_shld1_samp+pm_shld2_samp+ovrnight_samp;
```
## Maximum Free Flow speed
The strategy for computing free flow speed using the 8 pm to 5:30 am period turns out to have a few problems. In some cases, links have such small traffic volumes overnight that they don't have a free flow speed. In other cases, the free flow period speed ends up being slower than the speed on that link in other periods. By definition, free flow speeds should be the fastest average speeds on the link-- otherwise the metrics computed from it (benefits of congestion reduction, for example) would be misleading. Therefore the `max_ff_speed` and `max_ff_period` fields contain the maximum average speed on the link and the time period in which that speed occurs.

``` sql
update trafficdata.linkmedspd_2013
set max_ff_spd = greatest(ff_spd, am_shld1_spd,am_peak_spd, am_shld2_spd,midday_spd, pm_shld1_spd,pm_peak_spd,pm_shld2_spd, ovrnight_spd);


update trafficdata.linkmedspd_2013
set max_ff_period = case 
	when max_ff_spd=ff_spd then 'freeflow'
	when max_ff_spd=am_shld1_spd then 'am_shld1'
	when max_ff_spd=am_peak_spd then 'am_peak'
	when max_ff_spd=am_shld2_spd then 'am_shld2'
	when max_ff_spd=midday_spd then 'midday'
	when max_ff_spd=pm_shld1_spd then 'pm_shld1'
	when max_ff_spd=pm_peak_spd then 'pm_peak'
	when max_ff_spd=pm_shld2_spd then 'pm_shld2'
	when max_ff_spd=ovrnight_spd then 'overnight'
	else 'err'
	end;
```
## Planning Time Index
The planning time index is a measure of traffic predictability and is the ratio of the 95th percentile travel time to the median travel time. For more info on planning time, see http://www.ops.fhwa.dot.gov/publications/tt_reliability/ttr_report.htm.

Since we are computing planning time for each link, (meaning that travel distance is constant) the formula for planning time index is:

	median speed / 5th percentile speed

We used the median of the median (`am_perc_50_median` or `pm_perc_50_median`) and the 5th percentile of the median ( `am_perc_05_median` or `pm_perc_05_median`) to avoid outliers skewing the data. **Therefore, this statistic tells us how predictable the average morning or evening peak is.** 


``` sql
update trafficdata.linkmedspd_2013
set am_perc_05_median= am.perc_05_median,
    am_perc_50_median = am.perc_50_median,
    am_wtd_mean_05th= am.wtd_mean_05th
from linkspdampeak am
where am.link_id= linkmedspd_2013.link_id;

update trafficdata.linkmedspd_2013
set pm_perc_05_median= pm.perc_05_median,
    pm_perc_50_median = pm.perc_50_median,
    pm_wtd_mean_05th= pm.wtd_mean_05th
from linkspdpmpeak pm
where pm.link_id= linkmedspd_2013.link_id;	

update trafficdata.linkmedspd_2013
set am_pti = am_perc_50_median/am_perc_05_median,
    pm_pti = pm_perc_50_median/pm_perc_05_median;
```

The other option would be to use the ratio of weighted average of the median (`am_peak_spd` or `pm_peak_spd`) to the weighted average of the 5th percentile (`am_wtd_mean_05th` or `pm_wtd_mean_05th`, also contained in the table). This statistic, *not computed in the table but easily computable in arcgis or excel*, would show the predictability experienced on that link by the average vehicle. Note that this second measure would be much more sensitive to outliers and data errors. 

