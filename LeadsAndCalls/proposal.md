# Leads And Calls

The purpose of this document is to discuss alternatives regarding data processing and solving current pain points around our day-to-day operation for Leads and Calls.

## Summary
This are the main pain points over the leads and calls process we have:

- Currently we are not reporting _revenue from calls_ having more than one lead, this is causing issues in our revenue reports (not reporting expected revenue).
- Leads have several types of deals, cpa/cpt, warm transfer, drips, lead calls, and all are processed in and big complex process with lots of custom rules for every type.
- Disposition process for leads and calls is to complex and requires lot of maintainace.
- CPA / CPT Leads process is not standarized and it's custom for every CPA Broker which requires a lot of maintaince.
- CBM (Call Business Migration) process, wich involves leads and calls, requires a custom orchestration, and relies in Stored Procedure with the business logic, making it dependend on the database.

## Proposal Overview

We want to start tackling down the complexity of all this processes in order to reduce the maintaince of the leads and calls. The problem starts with having to consolidate the data in one big table that adapts to all the differente types of deals, and when new data is received (new sales, refunds, etc) we need to "reprocess" which consist in delete, consolidate again and insert, or well in scenarios like _disposition, cpa / cpt, warm transfer and calls_ we directly update the already consolidated data.

This mixed process between consolidation and updates is not only costly, it also provides another problem, when the data is updated by a process we are no longer able to reprocess the data, because all the treatments applied to the data are not part of the base consolidation process, this update processes are in separated pipelines that search and update the data. Those are executed with different rules according to the provider. The problem are those mixed processes over data materialized, but why we need to materialize the data, well is because of reporting and Aurora, we have to provide the consolidated data to reporting and when we materialized in just one table we ease the access to it. So when we need to add a field to that big table or simply recalculated the data over the new one provided this result in costly **operational process**.

So what we propose?, we want to find a work around for the materialization of the data and this is simple, do not materialize the data, rely on queries or views that refreshs every time the data is readed and stop building materialization processes or consolidation process as we call them. I know this sounds good at first, but when you think about it, for this to work we need to build big queries to do this, and doing this in MySql or Aurora is going to be costly, difficult but not imposible, but, if we do this in RedShift this is much more likely to be feasible. So we are proposing not only migrate to redshift but start to using view to "materialize" the data.

This is a big project that could reduce the maitaince time and effort to less than half of the time, because instead of constantly fixing ETL and reprocess, which takes a lot of time, we only update the query and done. 

The other problem is the time, this project is not posible do it, or at least do it right in just one quarter this probably would take more than one, so we are going to start this approach by giving baby steps. First we are going to start by separating concerns, as _Cristian_ said _"They are not billing us by table..."_ we are going to separate the big fat leads consolidation table (_boberdoo_ table in the _log_upload_ schema), into small tables like, transactions over leads (we already have this one), transactions over leads with cpa or cpt deal, transactions over warm transfers, transactions over lead calls, and so on, this will allow us to break the current big fat process into small ones with only one concern and it's rules. 

But what breaking into tables and smalls processes will do for us, well this will lead us to rely only in a process that consolidates all this tables, without updates, and this process will be a select/insert kind. This seems  something that eventually could evolve in just a select process.

## What we need to do

First we have 5 business units that are treated as one big fat process:

- Leads (plain and simple): this are the ones we send to boberdoo platform, and some broker may buy it, and this broker pay us for that lead.
- Leads with CPA/CPT deal: this leads are the ones that cpa cpt identified brokers buys (yea buys them, because they win the lead auction, but the revenue is simbolic), but only pays the ones the broker actually made a sale of it.
- CBM: this are calls, where the calls are registered in the invoca platform but campaigns and revenue are in boberdoo.
- Warm Transfer: leads that warm transferm deal brokers buys and pays us for the ones they sold, this sounds kind like CPA/CPT flow, it is but is a different kind of deal.
- Disposition: this is a big fat process, this process is to help us measure the quality of leads we are saling, in order to know this, we receive feedback from the brokers about the leads and calls we sold them and what is the status of that lead, if lead was sold or not, if is still on hold, etc. This is currently a very painful process beacuse the current implementation is way to complex.

All this 5 business units need to be processed and "consolidate" in the _boberdoo_ table and _invoca_raw_ table. By splitting these processes into small ones with every one with the necesary tables to allow us to consolidate just with a query.

## Roadmap