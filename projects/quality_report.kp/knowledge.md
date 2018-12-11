---
title: SF History in DWH
authors:
- hpvaleriy
tags:
- quality_report
- appointments
created_at: 2018-12-03 00:00:00
updated_at: 2018-12-04 14:58:40.235461
tldr: This is short description of the content and findings of the post.
---

**Contents**

[TOC]

_NOTE: this will include a table of contents when rendered on the site._



# Quality Report (Quality Steering)
The goal of the process in general and the report in particular is to understand how to make first appointments with partners more productive by providing data with better quality.

The process starts with weightings provided for different aspects of the customer profile.
They are different by country. At the moment we have DEU and CHE, there is a ticket to add NLD (https://audibene.atlassian.net/browse/BI-3112).

Tableau: https://intelligence.audibene.de/#/views/QualitySteering/Germany

In my example I will use **DEU**.

P.S. in the text I replace "_(double underscore)c" with "_c", because of the formatting issues.


## Excel source
The source for weightings is this Excel file(Sharepoint):
`INT Business Intelligence/incoming_data/sales/Quality_Appointments.xlsx`

[Web URL of the file](https://audibene.sharepoint.com/sites/INTBusinessIntelligence/_layouts/15/Doc.aspx?sourcedoc=%7BCBBF7F8B-E577-4BF5-9FEB-AA2FFA1D1981%7D&file=Quality_Appointments.xlsx&action=default&mobileredirect=true)

Any change to the file triggers [MS Flow](https://emea.flow.microsoft.com/manage/environments/Default-51834058-d59b-46ca-8d10-e86de9ee892d/flows/shared/1159911b-2958-42e7-a914-e6f029ec03fc/details) to run a Rundeck job: [DWH/Sales/quality_appointments](https://rundeck-bi.service-production.audibene.net/project/DWH/job/show/648d63a9-2766-4441-adda-f2838d493002).

In the file we have the first table of actual weightings:
* "DWH Key": text key that is used in DWH to represent the feature/weighting
* "Feature Name": the human-readable name of the feature
* "Comments": self-describing
* "Value": value in that represents %, e.g. 0.05 stands for 5%
* "Validation" section: a simple check to see if all % sum up to 100%

The second table with bracets for some features, like consulting_time:
* "DWH Key": text key that is used in DWH to represent the feature/weighting
* "Low Bracket": the low border of the bracket (inclusive)
* "High Bracket": the high border of the bracket (inclusive)
* "Value": any eligible value


## Staging layer
Ext2Stg ETL procedure is the part of the primary SF plan and consists of this:
* Rundeck job: [DWH/Sales/quality_appointments](https://rundeck-bi.service-production.audibene.net/project/DWH/job/show/648d63a9-2766-4441-adda-f2838d493002)
* PostgreSQL function is: `stg_sales.sp_ext2stg_appointment_quality`
* DWH tables is: `stg_sales.appointment_quality`

#### Current features

##### Hearing test
Possible values are: 0 - no hearing test performed, 2 - there was a hearing test, -1 - a lead is not eligible for a hearing test.
I apply these filter to determine eligibility(the source is stg_sales.salesforce_hearing_test_c):
* status in ('qualified', 'qualifiziert')
* coalesce(prescription_c, 'Unknown') in ('Liegt nicht vor', '-', 'No', 'Not availabe', 'Not available', 'Unknown') - Not applied to CHE
* type_of_treatment_c in ('First care', 'First-time user', 'Erstversorgung', 'Firstcare')
* coalesce(lead_closing_category_c, '') not in ('Provider', 'Technical')

##### Consultation time
Possible values are: any value above 0 - represents call_time_minutes(formula is `1.0 * sum(call_time_seconds) / 60`), -1 - there was no calls with type = `First Appts Call` associated with a lead/opportunity
The source is `stg_sales.call`.

##### Product recommendation
Possible values are: 0 - no product recommendation was made, 2 - there was a product recommendation, -1 - partner is not eligible for this feature.
DEU source is: `stg_sales.salesforce_account` (with a few filters applied), also I check `stg_sales.salesforce_opportunity.discussed_chip_platform_device`
CHE source is: `stg_sales.salesforce_productrecommendation_c` - I look for brands of partners, also I check `stg_sales.salesforce_opportunity.manufacturer_recommendation_c`

##### Days to the first appointment
Possible values are: any value above 0 - represents amount of days between opportunity created date and the first appointment of the customer(if it took place already then customer showed up, if not the value could change over time, because the customer could cancel/reschedule the appointment any time), -1 - the source field is null
The source is `stg_sales.salesforce_opportunity.time_to_first_appointment_c`.

##### Profile Depth (only DEU, for now)
There is a weight for this feature itself, but it actually consists of more smaller features, the current list is:
first_second_name_filled, phone_filled, mobile_filled, email, former_profession, professional_status, glasses, customer_type, type_of_insurance, preliminary_insurance, cosi1, cosi2, cosi3, type_of_treatment, recommendation_ent, degree_of_suffering, motivational_level, reason_for_now, tinnitus, pacemaker, length_general_notes, length_medical_notes
All of these sub-features/weightings are simple boolean flags whether the value is filled or not, except length_general_notes & length_medical_notes.

_length_general_notes_ - it's percentile from 0 to 100 with step of 1, i.e. 0,1,2,3..98,99,100. It's based on length of the text of the field special_notes_fitting_profile_c(stg_sales.salesforce_opportunity).

_length_medical_notes_ - it's percentile from 0 to 100 with step of 1, i.e. 0,1,2,3..98,99,100. It's based on length of the text of the field medical_comments_c(stg_sales.salesforce_opportunity).

For CHE _length_general_notes_ is used as a separate feature and not as part of _profile_depth_.
The formula is: `length(special_notes_fitting_profile_c) / AVG(length(special_notes_fitting_profile_c))` for this week.
Possible values are from 0 to eternity :)

_P.S.The window or set of opportunities to be executed based on is always a week of the current opportunity._

##### Simple boolean fields
Those are: date_of_birth, address


## Datamart layer

#### dim_opportunity
Source to fill the dimension is in "db/datamart/etl/dim/0120 - sp_dim_opportunity.sql".
I fill column is_qualitative in the modules "appointment_quality_deu", "appointment_quality_che"

For all the features except profile_depth I do as follows:
1. Determine the base for division by checking if any feature value is -1 (means feature shouldn't be considered at all).
2. For example, by default sum of all the features weightings is 1 (i.e. 100%)
3. If a lead was not eligible for a hearing test, the feature value of hearing_test will be -1. The feature's weighting is 0.125(i.e. 12.5%).
4. The new base for division is (1-0.125) = 0.875 (i.e. 87.5%)
5. The default threshold is 60
6. We multiply 60 by 0.875 => 52.5 is the new threshold

For profile_depth I calculate sum of all the sub-features values and then calculate the percentile, multiply by 100 and multiply by the feature(profile_depth) value.

Example: `sum_of_sub_features` = 80, `percentile` = 0.7, `result` = 0.7 * 100 * 0.15 (profile_depth current value) => 10.5

#### fact_consultant_daily
Columns that use dim_opportunity.is_qualitative are: ocd_appts_qualitative, ocd_appts_qualitative_filled

## Reporting layer



## Detailed overview
Overview of the features per opportunity: `dev_val.vw_qualitative_appointments_details` (still in Dev.)

https://audibene.atlassian.net/browse/BI-2952