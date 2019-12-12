```
options center nodate nonumber ps=60 ls=72;
dm'log;clear;out;clear;';
ods html close; /* close previous */
ods html; /* open new */


Data cbm;
input Name & $ 30.	Year	Make $	Model $	Mile	MeanMPG	CleanBoost $;
datalines;
Albrecht Bolivar 	1997	Jag	XJR	1448	17.2	No
Luis Contreras	 2008	Chevy	Impala	1565	21.4	No
Franklin Bond	 2008	Honda	Accord	2761.6	26.5	No
Jovush 	2003	Honda	Accord	732	29	No
Albrecht Bolivar 	1996	Jag	XJ6	2715	21.4	No
Daniel Foerstner 	2012	Honda	Civic	889	35.5	No
Joe Zweng	 2010	Toyota	Camry	669	23.9	No
Robert Reolle 	2014	Chevy	Cruise	2042	30.8	No
James Tbagg	 1998	Chevy	Tahoe	730	13.1	No
Chris Miller 	2004	Mazda	Mazda6	2365	27.6	No
Tyler Phelps 	2001	Dodge	Ram1500	1182	12.5	No
Halelorf	 2000	Lincoln	Towncar	1118	25.8	No
Dieselpower191 	2014	Ford	F150	1797	12	No
Ray Demafiles	 2003	Toyota	Corolla	2039	33.1	No
Joe Michieli	 2009	Mazda	RX8	758	14.3	No
Chris Snyder	 2006	Toyota	Prius	1956	49.4	No
John Vickery	 1991	Mazda	626	1203	33.6	No
Gonzalez	 2011	Dodge	Challenger	1394	22.4	No
Big Guy Review 	2007	Ford	E250	5659	16.8	No
Austin Boni	 2007	Volvo	S40T5	1837	21.1	No
Albrecht Bolivar 	1997	Jag	XJR	778	15.9	Yes
Luis Contreras	 2008	Chevy	Impala	760	20.6	Yes
Franklin Bond	 2008	Honda	Accord	3010.3	25.9	Yes
Jovush	 2003	Honda	Accord	713	28.5	Yes
Albrecht Bolivar 	1996	Jag	XJ6	722	21.4	Yes
Daniel Foerstner 	2012	Honda	Civic	655	35.5	Yes
Joe Zweng	 2010	Toyota	Camry	658	24	Yes
Robert Reolle	 2014	Chevy	Cruise	2042	31	Yes
James Tbagg	 1998	Chevy	Tahoe	935	13.5	Yes
Chris Miller 	2004	Mazda	Mazda6	1160	28.1	Yes
Tyler Phelps 	2001	Dodge	Ram1500	314	13.1	Yes
Halelorf	 2000	Lincoln	Towncar	1335	26.5	Yes
Dieselpower191 	2014	Ford	F150	2000	12.7	Yes
Ray Demafiles 	2003	Toyota	Corolla	2058	34	Yes
Joe Michieli 	2009	Mazda	RX8	876	15.3	Yes
Chris Snyder 	2006	Toyota	Prius	967	50.6	Yes
John Vickery 	1991	Mazda	626	1207	35.4	Yes
Gonzalez	 2011	Dodge	Challenger	1480	24.4	Yes
Big Guy Review 	2007	Ford	E250	4013.5	19.1	Yes
Austin Boni	 2007	Volvo	S40T5	1971	32.5	Yes
;

/*Creates dataset of no CleanBoost Maxx*/
data cbmn;
set cbm;
if CleanBoost = 'No';
MeanMPGNo=MeanMPG;
run;


/*Creates dataset with CleanBoost Maxx*/
data cbmy;
set cbm;
if CleanBoost = 'Yes';
MeanMPGYes=MeanMPG;
Keep MeanMPGYes;
run;

data cbmpairt;
merge cbmn cbmy;
diff = MeanMPGYes - MeanMPGNo;
Percentchange = diff/MeanMPGNo;
run;

proc print data=cbm;
title 'CleanBoost Maxx MPG Experiemnt';
run;
/*Check normality of differences Shapiro Wilks. Good if P-Value > .05 */
/* H0: Data is normal
   Ha: Data is not normal
*/
proc univariate normal data=cbmpairt;
var diff percentchange;
histogram;
run;
/* not normal. Cannot proceed. Use Wilcoxan Sign Rank non-Parametric*/

/*Check Constant Variance. Good is P-Value > .05 */
/*Not needed for pair t test */
/* H0: VarianceMPGYes = VarianceMPGNo 
   Ha: VarianceMPGYes =/= VarianceMPGNo 
*/
/*
proc glm data=cbm;
class CleanBoost;
model MeanMPG = CleanBoost;
means CleanBoost / hovtest;
run;
*/



/*taking out outlier Volvo Outlier*/
data cbmpairtnovolvo;
set cbmpairt;
if Make = 'Volvo' then delete;
run;

/*Check normality of differences Shapiro Wilks. Good if P-Value > .05 */
/* H0: Data is normal
   Ha: Data is not normal
*/
proc univariate normal data= cbmpairtnovolvo;
var diff percentchange;
histogram;
run;
/*Differences are normal without Volvo*/

/*Pair t test. P-Value < .05 to be significantly different*/
/* H0: MeanMPGYes-MeanMPGNo = 0
   Ha: MeanMPGYes-MeanMPGNo =/= 0
*/
proc ttest data=cbmpairtnovolvo;
paired MeanMPGYes*MeanMPGNo;
run;
/* With Volvo Outlier Removed, there is significant difference */

/* T-test of percentchange */
/* H0: Miu = 0
   Ha: Miu =/= 0 
*/
proc ttest data=cbmpairtnovolvo alpha=.05 H0=0 ;
var percentchange;
run;
/* P-Value = .0478
   barely significant difference. Reject H0, and conclude that there is difference.

/* Let's see if CleanBoost Maxx Increases MPG using t test again on PercentChange */
/* H0: Miu = 0
   Ha: Miu > 0 
*/
proc ttest data=cbmpairtnovolvo alpha=.05 H0=0 sides=u;
var percentchange;
run;
/* P-Value = .0244
   Reject H0 and conclude Ha that MPG increases after applying CleanBoost MAxx */
 ```
