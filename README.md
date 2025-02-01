# Regression-and-Machine-Learning-Models
/*
(1) import local files: 2020_HMDA_Final.csv, 2021_HMDA_Final.csv, 2022_HMDA_Final.csv

fix format:
Appdate
Age,coa_age, DTI, CLTV, CreditScore,coa_CreditScore,loan_term, PercMinor, PercMedian


NOTE: The data set WORK.'2020_HMDA_FINAL'n has 46,779 observations and 231 variables.
NOTE: The data set WORK.'2021_HMDA_FINAL'n has 39,006 observations and 230 variables.
NOTE: The data set WORK.'2022_HMDA_FINAL'n has 17,707 observations and 222 variables.

*/
options compress=yes nocenter nodate;
Data WORK.HMDA_Final_PBG_2020_2022;
	set WORK.'2020_HMDA_FINAL'n(drop= 'count.x'n 'count.y'n Asian_Perc Black_Perc Hispanic_Perc)
		WORK.'2021_HMDA_FINAL'n(drop= 'count.x'n 'count.y'n Asian_Perc Black_Perc Hispanic_Perc) 
		WORK.'2022_HMDA_FINAL'n(drop= 'count.x'n 'count.y'n Asian_Perc Black_Perc Hispanic_Perc);
	FB_year = year(ActionDate);
	rename Younger_YN=Age_LT62_YN Older_YN=Age_GE62_YN;
run;

proc contents data=WORK.HMDA_Final_PBG_2020_2022 varnum; run;


%macro ns(sheetname);
	ods excel options(sheet_interval="table");
	ods exclude all;
	data _null_; file print; put _all_;run;
	ods select all;
	ods excel options(zoom='80' sheet_interval="None" sheet_name="&sheetname");
%mend;

%macro odstxt(mytxt);
	proc odstext; p "&mytxt" / style = [color=red font_weight=bold]; run;
%mend;

%macro t(row, col);
	table All &row, (All &col)*N*f=comma8. / nocellmerge box="&row";
%mend;
/***********************************************************************************/


data WORK.HMDA_Final_PBG_2020_2022_fb;
set WORK.HMDA_Final_PBG_2020_2022;
	if LOS = "EN" or entity="EN" then FB_WMVF = "WM";
	else if LOS = "VF" or entity = "Byte" then FB_WMVF = "VF";
	else if LOS = "Portfolio" or entity = "Portfolio" then FB_WMVF="Portfolio";
	else FB_WMVF = cat(LOS, Entity);
run;

data WORK.HMDA_Final_PBG_2020_2022_WMVF;
set WORK.HMDA_Final_PBG_2020_2022_fb;
	if FB_WMVF in ("VF", "WM");
run;

%let myfile = WORK.HMDA_Final_PBG_2020_2022_fb;
/*
White_Non_Hispanic_YN
Black_Final_YN
Asian_Final_YN
AmercInd_Final_YN
PacIsland_Final_YN
Hispanic_Final_YN
Sex_Combined
Male_YN
Female_YN
Younger_YN
Older_YN
*/


proc format ;
	value $GC_sent
		"NO" = "NO"
		"No" = "NO"
		"no" = "NO"
		other = "YES";

	/*value myfmt 
		.='N/A'
		other = 'Valid';*/
run;


ods excel file="Contents_HMDA_Final_PBG_2020_2022.xlsx" options(sheet_interval="none");
	proc contents data=&myfile varnum;run;
%ns(Distribution);
	proc freq data=&myfile;
		tables 
			FB_year*(ApplDate  ActionDate)
			OccupancyType
			Action
			LoanType
			LoanType*LoanGroup
			Bank
			Bank*FB_WMVF*LOS*entity

			LOS
			FB_year*LOS
			FB_year*LOS*LoanType
			FB_year*LOS*entity
			FB_year*FB_WMVF*LOS*entity

			FB_year*FB_WMVF*LoanType

			Purpose
			Lien_Status
			/*LOB - not in 2022*/
			Preapproval
			ConstructionMethod
			
			STATE_ABRV
			MSA

			White_Non_Hispanic_YN
			Black_Final_YN
			Asian_Final_YN
			AmercInd_Final_YN
			PacIsland_Final_YN
			Hispanic_Final_YN

			Ethnicity_Combined*White_Non_Hispanic_YN*Hispanic_Final_YN
			
			Male_YN
			Female_YN
			Sex_Combined*Male_YN*Female_YN

			Age_LT62_YN
			Age_GE62_YN

		/missing list;
	format ApplDate  ActionDate yymon.;
	run;
%ns(Race_Dist);
proc freq data=&myfile;
	table
	White_Non_Hispanic_YN*Black_Final_YN*Hispanic_Final_YN*Asian_Final_YN*AmercInd_Final_YN*PacIsland_Final_YN	
	Race*CoaRace*(White_Non_Hispanic_YN 	Black_Final_YN	Hispanic_Final_YN	Asian_Final_YN 	AmercInd_Final_YN	PacIsland_Final_YN)	
	Race_1*Race_2*Race_3*Race_4*Race_5*CoaRace_1*CoaRace_2*CoaRace_3*CoaRace_4*CoaRace_5*(White_Non_Hispanic_YN Black_Final_YN	Hispanic_Final_YN	Asian_Final_YN 	AmercInd_Final_YN	PacIsland_Final_YN)
	/missing list;
run;

proc tabulate data=&myfile missing;
	class
	Race	Race_1	Race_2	Race_3	Race_4	Race_5 	
	/*Race1_Other		Race27_Other	Race44_Other*/
	CoaRace	CoaRace_1 CoaRace_2 CoaRace_3	CoaRace_4 CoaRace_5 
	/*CoaRace1_Other CoaRace27_Other	CoaRace44_Other*/

	White_Non_Hispanic_YN 	Black_Final_YN	Hispanic_Final_YN	Asian_Final_YN
	AmercInd_Final_YN	PacIsland_Final_YN	;

	%t(Race_1*Race_2*Race_3*Race_4*Race_5*Race, All);
	%t(CoaRace_1*CoaRace_2*CoaRace_3*CoaRace_4*CoaRace_5*CoaRace, All);

	%t(Race_1*Race_2*Race_3*Race_4*Race_5*CoaRace_1*CoaRace_2*CoaRace_3*CoaRace_4*CoaRace_5,
	White_Non_Hispanic_YN 	Black_Final_YN	Hispanic_Final_YN	Asian_Final_YN
	AmercInd_Final_YN	PacIsland_Final_YN);
run;

%ns(Eth_Dist);
proc freq data=&myfile;
	table
	Ethnicity_Combined*Ethnicity*Coa_Ethnicity
	Ethnicity*Coa_Ethnicity Ethnicity_1*Ethnicity_2*Ethnicity_3*Ethnicity_4*Ethnicity_5*Coa_Ethnicity_1*Coa_Ethnicity_2*Coa_Ethnicity_3*Coa_Ethnicity_4*Coa_Ethnicity_5*(White_Non_Hispanic_YN Hispanic_Final_YN)
	/missing list;
run;

proc tabulate data=&myfile missing;
	class
		Ethnicity_1	Ethnicity_2	Ethnicity_3	Ethnicity_4	Ethnicity_5	
	/*EthnicityOther*/	Coa_Ethnicity_1	
	Coa_Ethnicity_2	Coa_Ethnicity_3	Coa_Ethnicity_4	Coa_Ethnicity_5	/*Coa_EthnicityOther*/
	 White_Non_Hispanic_YN Hispanic_Final_YN	;

	%t(Ethnicity_1*Ethnicity_2*Ethnicity_3*Ethnicity_4*Ethnicity_5*Coa_Ethnicity_1*Coa_Ethnicity_2*Coa_Ethnicity_3*Coa_Ethnicity_4*Coa_Ethnicity_5	,
		 White_Non_Hispanic_YN Hispanic_Final_YN);
run;

%ns(Gen_Age_Dist);
proc tabulate data=&myfile missing;
	class
	Sex	CoaSex 	Sex_Combined
	Male_YN	Female_YN Age_LT62_YN Age_GE62_YN;

	%t(Sex*CoaSex*Sex_Combined, All);
	%t(Sex*CoaSex*Sex_Combined, Male_YN	Female_YN);
	%t(Age_LT62_YN*Age_GE62_YN, All);
run;
ods excel close;





2_WMVF_distribution

/*
White_Non_Hispanic_YN
Black_Final_YN
Asian_Final_YN
AmercInd_Final_YN
PacIsland_Final_YN
Hispanic_Final_YN
Sex_Combined
Male_YN
Female_YN
Younger_YN
Older_YN
*/

%let myfile = WORK.HMDA_Final_PBG_2020_2022_WMVF;


proc format ;
	value $GC_sent
		"NO" = "NO"
		"No" = "NO"
		"no" = "NO"
		other = "YES";

	/*value myfmt 
		.='N/A'
		other = 'Valid';*/
run;


title color="red" height=15pt bold /*italic*/ underline=1 "WMVF Contents";
	proc contents data=&myfile varnum;run;
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF Distribution";
	proc freq data=&myfile;
		tables 
			FB_year*(ApplDate  ActionDate)
			OccupancyType
			Action
			LoanType
			LoanType*LoanGroup
			Bank
			Bank*FB_WMVF*LOS*entity

			LOS
			FB_year*LOS
			FB_year*LOS*LoanType
			FB_year*LOS*entity
			FB_year*FB_WMVF*LOS*entity

			FB_year*FB_WMVF*LoanType

			Purpose
			Lien_Status
			/*LOB - not in 2022*/
			Preapproval
			ConstructionMethod
			
			STATE_ABRV
			MSA

			White_Non_Hispanic_YN
			Black_Final_YN
			Asian_Final_YN
			AmercInd_Final_YN
			PacIsland_Final_YN
			Hispanic_Final_YN

			Ethnicity_Combined*White_Non_Hispanic_YN*Hispanic_Final_YN
			
			Male_YN
			Female_YN
			Sex_Combined*Male_YN*Female_YN

			Age_LT62_YN
			Age_GE62_YN

		/missing list;
	format ApplDate  ActionDate yymon.;
	run;
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF Race";
proc freq data=&myfile;
	table
	White_Non_Hispanic_YN*Black_Final_YN*Hispanic_Final_YN*Asian_Final_YN*AmercInd_Final_YN*PacIsland_Final_YN	
	Race*CoaRace*(White_Non_Hispanic_YN 	Black_Final_YN	Hispanic_Final_YN	Asian_Final_YN 	AmercInd_Final_YN	PacIsland_Final_YN)	
	Race_1*Race_2*Race_3*Race_4*Race_5*CoaRace_1*CoaRace_2*CoaRace_3*CoaRace_4*CoaRace_5*(White_Non_Hispanic_YN Black_Final_YN	Hispanic_Final_YN	Asian_Final_YN 	AmercInd_Final_YN	PacIsland_Final_YN)
	/missing list;
run;

proc tabulate data=&myfile missing;
	class
	Race	Race_1	Race_2	Race_3	Race_4	Race_5 	
	/*Race1_Other		Race27_Other	Race44_Other*/
	CoaRace	CoaRace_1 CoaRace_2 CoaRace_3	CoaRace_4 CoaRace_5 
	/*CoaRace1_Other CoaRace27_Other	CoaRace44_Other*/

	White_Non_Hispanic_YN 	Black_Final_YN	Hispanic_Final_YN	Asian_Final_YN
	AmercInd_Final_YN	PacIsland_Final_YN	;

	%t(Race_1*Race_2*Race_3*Race_4*Race_5*Race, All);
	%t(CoaRace_1*CoaRace_2*CoaRace_3*CoaRace_4*CoaRace_5*CoaRace, All);

	%t(Race_1*Race_2*Race_3*Race_4*Race_5*CoaRace_1*CoaRace_2*CoaRace_3*CoaRace_4*CoaRace_5,
	White_Non_Hispanic_YN 	Black_Final_YN	Hispanic_Final_YN	Asian_Final_YN
	AmercInd_Final_YN	PacIsland_Final_YN);
run;

title color="red" height=15pt bold /*italic*/ underline=1 "WMVF Ethnicity";
proc freq data=&myfile;
	table
	Ethnicity_Combined*Ethnicity*Coa_Ethnicity
	Ethnicity*Coa_Ethnicity Ethnicity_1*Ethnicity_2*Ethnicity_3*Ethnicity_4*Ethnicity_5*Coa_Ethnicity_1*Coa_Ethnicity_2*Coa_Ethnicity_3*Coa_Ethnicity_4*Coa_Ethnicity_5*(White_Non_Hispanic_YN Hispanic_Final_YN)
	/missing list;
run;

proc tabulate data=&myfile missing;
	class
		Ethnicity_1	Ethnicity_2	Ethnicity_3	Ethnicity_4	Ethnicity_5	
	/*EthnicityOther*/	Coa_Ethnicity_1	
	Coa_Ethnicity_2	Coa_Ethnicity_3	Coa_Ethnicity_4	Coa_Ethnicity_5	/*Coa_EthnicityOther*/
	 White_Non_Hispanic_YN Hispanic_Final_YN	;

	%t(Ethnicity_1*Ethnicity_2*Ethnicity_3*Ethnicity_4*Ethnicity_5*Coa_Ethnicity_1*Coa_Ethnicity_2*Coa_Ethnicity_3*Coa_Ethnicity_4*Coa_Ethnicity_5	,
		 White_Non_Hispanic_YN Hispanic_Final_YN);
run;

title color="red" height=15pt bold /*italic*/ underline=1 "WMVF Gender Age";
proc tabulate data=&myfile missing;
	class
	Sex	CoaSex 	Sex_Combined
	Male_YN	Female_YN Age_LT62_YN Age_GE62_YN;

	%t(Sex*CoaSex*Sex_Combined, All);
	%t(Sex*CoaSex*Sex_Combined, Male_YN	Female_YN);
	%t(Age_LT62_YN*Age_GE62_YN, All);
run;






2_WMVF_1_ConvFHA_2_FHA

%let myfile = WORK.HMDA_Final_PBG_2020_2022_WMVF;
%let racelist = White_Non_Hispanic_YN Black_Final_YN Hispanic_Final_YN Asian_Final_YN AmercInd_Final_YN PacIsland_Final_YN;
%let ethlist = White_Non_Hispanic_YN Hispanic_Final_YN;
%let genagelist = Male_YN Female_YN	Age_LT62_YN Age_GE62_YN;

title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Applications 1-5";
	proc tabulate data=&myfile missing;
		class loantype &racelist &ethlist &genagelist;
		%t(&racelist, loantype);
		%t(&ethlist, loantype);
		%t(&genagelist, loantype);
	where loantype in (1,2) and Action in (1, 2, 3,4,5);
	run;
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2";
	proc tabulate data=&myfile missing;
		class loantype &racelist &ethlist &genagelist;
		%t(&racelist, loantype);
		%t(&ethlist, loantype);
		%t(&genagelist, loantype);
	where loantype in (1,2) and Action in (1, 2);
	run;
title;

title color="red" height=15pt bold /*italic*/ underline=1 "FHA VF-WM Applications 1-5";
	proc tabulate data=&myfile missing;
		class FB_WMVF &racelist &ethlist &genagelist;
		%t(&racelist, FB_WMVF);
		%t(&ethlist, FB_WMVF);
		%t(&genagelist, FB_WMVF);
	where loantype in (2) and Action in (1, 2, 3,4,5);
	run;
title color="red" height=15pt bold /*italic*/ underline=1 "FHA VF-WM Approval 1-2 Gender Deep Dive";
	proc tabulate data=&myfile missing;
		class FB_WMVF &racelist &ethlist &genagelist FB_year;
		%t(&racelist, FB_WMVF );
		%t(&ethlist, FB_WMVF);
		%t(&genagelist, FB_WMVF);
		%t(&genagelist, FB_year*FB_WMVF);
	where loantype in (2) and Action in (1, 2);
	run;
title;



2_WMVF_1_ConvFHA - model var & overall

%macro lg(tar, var);
	proc logistic data=&myfile;
	class &var (ref='0');
	model &tar (event='2') = &var      / expb;
	where &tar in (1,2) and Action in (1, 2) and (White_Non_Hispanic_YN=1 or &var=1);
	run;
%mend;


data WORK.HMDA_Final_PBG_2020_2022_WMVF_m;
set WORK.HMDA_Final_PBG_2020_2022_WMVF;

/* +++ vars: 
purpose, 
OccupancyType


FL_DTI_bin
FL_CLTV_bin
FL_FICO_bin

FL_units
FL_jumbo
FL_property_group

*/


length FL_DTI_bin $ 8;
if /* this has only 8 record - DTIRatio=. then FL_DTI_bin = "missing";*/
/* this has only 1 record - else if .<DTIRatio<0 then FL_DTI_bin = "<0";*/
DTIRatio<=43 then FL_DTI_bin = "<=43";
else if 43<DTIRatio<=50 then FL_DTI_bin = "43-=50";
else if 50<DTIRatio then FL_DTI_bin = ">50";
else FL_DTI_bin="other";


length FL_CLTV_bin $ 9;
if CLTV=. then FL_CLTV_bin = "missing";
else if .<CLTV<=0 then FL_CLTV_bin = "<=0";
else if 0<CLTV<=75 then FL_CLTV_bin = "0-=75";
else if 75<CLTV<=80 then FL_CLTV_bin = "75-=80";
else if 80<CLTV<=95 then FL_CLTV_bin = "80-=95";
else if 95<CLTV<=100 then FL_CLTV_bin = "95-=100";
else if 100<CLTV<=105 then FL_CLTV_bin = "100-=105";
else if 105<CLTV then FL_CLTV_bin = ">105";
else FL_CLTV_bin="other";


FL_FICO = CreditScore;
if coarace_1=8 then FL_FICO = min(CreditScore, Coa_CreditScore);

length FL_FICO_bin $ 8;
if FL_FICO<=620 then FL_FICO_bin = "<=620";
else if 620<FL_FICO<=640 then FL_FICO_bin = "620-=640";
else if 640<FL_FICO<=680 then FL_FICO_bin = "640-=680";
else if 680<FL_FICO<=700 then FL_FICO_bin = "680-=700";
else if 700<FL_FICO<=740 then FL_FICO_bin = "700-=740";
else if 740<FL_FICO then FL_FICO_bin = ">740";
/*else FL_FICO_bin="other";*/

FL_units = "single";
if TotalUnits >1 then FL_units="multi";

FL_jumbo = "Conforming";
if LoanAmount > 647.2 then FL_jumbo="Jumbo";

length FL_property_group $ 9;
if property_group=' ' then FL_property_group="NA";
else FL_property_group=property_group;

run;


%let myfile = WORK.HMDA_Final_PBG_2020_2022_WMVF_m;
/***
Model
***/
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: overall model var ";
	proc freq data=&myfile;
		tables loantype action purpose OccupancyType FL_DTI_bin FL_CLTV_bin FL_FICO_bin FL_units FL_jumbo FL_property_group
		/missing list;
	where loantype in (1,2) and Action in (1, 2);
	run;

title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: overall model var by loantype";
	proc freq data=&myfile;
		tables (action purpose OccupancyType FL_DTI_bin FL_CLTV_bin FL_FICO_bin FL_units FL_jumbo FL_property_group)*loantype
		/missing list;
	where loantype in (1,2) and Action in (1, 2);
	run;
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: overall model estimate";
	%let modelvars = purpose OccupancyType FL_DTI_bin FL_CLTV_bin FL_FICO_bin FL_units FL_jumbo FL_property_group;
	proc logistic data=&myfile;
	class purpose OccupancyType FL_DTI_bin(ref='<=43') FL_CLTV_bin(ref='75-=80') FL_FICO_bin(ref='>740') FL_units FL_jumbo FL_property_group / Param=glm;
	model loantype (event='2') = &modelvars  / expb;
	/*roc;*/
	ods output ROCassociation=overall_model_ROCassociation;
	where loantype in (1,2) and Action in (1, 2);
	run;
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: overall model AUC";
	proc print data=overall_model_ROCassociation;run;








# 2_WMVF_1_ConvFHA-model pbg
 %let myfile = WORK.HMDA_Final_PBG_2020_2022_WMVF_m;


title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: Model PBG - RAW";
	%lg(loantype, Black_Final_YN);
	%lg(loantype, Hispanic_Final_YN);
	%lg(loantype, Asian_Final_YN);
	%lg(loantype, AmercInd_Final_YN);
	%lg(loantype, PacIsland_Final_YN);
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: Model PBG - RAW";
	proc logistic data=&myfile;
	class Female_YN(ref='0') / Param=glm;
	model loantype (event='2') = Female_YN      / expb;
	where loantype in (1,2) and Action in (1, 2) and (Male_YN=1 or Female_YN=1);
	run;
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: Model PBG - RAW";
	proc logistic data=&myfile;
	class Age_GE62_YN(ref='0') / Param=glm;
	model loantype (event='2') = Age_GE62_YN      / expb;
	where loantype in (1,2) and Action in (1, 2) and (Age_LT62_YN=1 or Age_GE62_YN=1);
	run;



/***
Model PBG output
***/

%let modelvars = purpose OccupancyType FL_DTI_bin FL_CLTV_bin FL_FICO_bin FL_units FL_jumbo FL_property_group;
%macro lg_var1(tar, var);
	proc logistic data=&myfile;
	class purpose OccupancyType FL_DTI_bin(ref='<=43') FL_CLTV_bin(ref='75-=80') FL_FICO_bin(ref='>740') 
		FL_units FL_jumbo FL_property_group &var(ref='0') / Param=glm;
	model &tar (event='2') = &modelvars &var      / expb;
	where &tar in (1,2) and Action in (1, 2) and (White_Non_Hispanic_YN=1 or &var=1);
	run;
%mend;

title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: Model PBG - Race modeled";
	%lg_var1(loantype, Black_Final_YN);
	%lg_var1(loantype, Hispanic_Final_YN);
	%lg_var1(loantype, Asian_Final_YN);
	proc logistic data=&myfile;
	class purpose OccupancyType FL_DTI_bin(ref='<=43') FL_CLTV_bin(ref='75-=80') FL_FICO_bin(ref='>740') 
		FL_units FL_jumbo FL_property_group AmercInd_Final_YN(ref='0') / Param=glm;
	model loantype (event='2') = &modelvars AmercInd_Final_YN      / expb;
	output out=WORK.WMVF_ConvFHA_Amerind_Modeled p=FB_pred;
	where loantype in (1,2) and Action in (1, 2) and (White_Non_Hispanic_YN=1 or AmercInd_Final_YN=1);
	run;
	%lg_var1(loantype, PacIsland_Final_YN);
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: Model PBG - Gender modeled";
	proc logistic data=&myfile;
	class purpose OccupancyType FL_DTI_bin(ref='<=43') FL_CLTV_bin(ref='75-=80') FL_FICO_bin(ref='>740') 
		FL_units FL_jumbo FL_property_group Female_YN(ref='0') / Param=glm;
	model loantype (event='2') = &modelvars Female_YN      / expb;
	where loantype in (1,2) and Action in (1, 2) and (Male_YN=1 or Female_YN=1);
	run;
title color="red" height=15pt bold /*italic*/ underline=1 "WMVF FHA-Conv Approval 1-2: Model PBG - Age modeled";
	proc logistic data=&myfile;
	class purpose OccupancyType FL_DTI_bin(ref='<=43') FL_CLTV_bin(ref='75-=80') FL_FICO_bin(ref='>740') 
		FL_units FL_jumbo FL_property_group Age_GE62_YN(ref='0') / Param=glm;
	model loantype (event='2') = &modelvars Age_GE62_YN      / expb;
	output out=WORK.WMVF_ConvFHA_Age_Modeled p=FB_pred;
	where loantype in (1,2) and Action in (1, 2) and (Age_LT62_YN=1 or Age_GE62_YN=1);
	run;







