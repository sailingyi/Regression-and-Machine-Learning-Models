import pandas as pd
import statsmodels.api as sm #- not good
#from sklearn.linear_model import LogisticRegression
from openpyxl import Workbook
import os
import numpy as np

UW_file_path = "Z://Shared/Compliance/Fair & Responsible Banking/Fair Lending Reviews/2024-2025Q2 Mortgage Steering Review/Regression Analysis/Data"
UW_file_name = "2024_2025Q2_WM ConvFHA_UW_Data_ModelVar.xlsx"
inputfile = os.path.join(UW_file_path, UW_file_name)
Out_path = "Z://Shared/Compliance/Fair & Responsible Banking/Fair Lending Reviews/2024-2025Q2 Mortgage Steering Review/Regression Analysis/"

#FL_target = "LoanType"
FL_target = "LOANTYPE"
#FL_PBG_All = ["White_Non_Hispanic_YN", "AmercInd_Final_YN", "Asian_Final_YN", "Black_Final_YN", 
#                    "Hispanic_Final_YN", "PacIsland_Final_YN",
#                    "Male_YN", "Female_YN", 'Younger_YN','Older_YN']
FL_PBG_All = ["White_Non_Hispanic_YN", "AmercInd_Final_YN", "Asian_Final_YN", "Black_Final_YN", 
                    "Hispanic_Final_YN", "PacIsland_Final_YN",
                    "Gender_Control_YN","Gender_Target_YN", "Age_Control_YN", "Age_Target_YN"]
# PBG
FL_PBG_RaceEth = ["White_Non_Hispanic_YN", "AmercInd_Final_YN", "Asian_Final_YN", "Black_Final_YN", 
                    "Hispanic_Final_YN", "PacIsland_Final_YN"]
#FL_PBG_Gen = ["Male_YN","Female_YN"]
FL_PBG_Gen = ["Gender_Control_YN","Gender_Target_YN"]	

#FL_PBG_Age = ["Younger_YN", "Older_YN"]
FL_PBG_Age = ["Age_Control_YN", "Age_Target_YN"]	

X_features = ["DTI_LessEq43",
"DTI_Great50_orNegative_orMissing",
"CLTV_Great75_LessEq80",
"CLTV_Great80_LessEq95",
"CLTV_Great95_LessEq100",
"CLTV_Great100_LessEq105",
"CLTV_Great105",
"CLTV_Missing",
"RT_Refinance",
"CashOut_Refinance",
"Second_Home",
"Investment_Property",
"Manufactured_Home",
"Condo",
"Other_Property",
"Cred_GreatEq620_Less640",
"Cred_GreatEq640_Less680",
"Cred_GreatEq680_Less700",
"Cred_GreatEq700_Less740",
"Cred_GreatEq740",
"Cred_Missing",
"Cred_NA",
"LoanAmount_GreatConforming",
"Multi_Unit"
]


try:
    UW = pd.read_excel(inputfile)
    for i in range(len(FL_PBG_All)):
        if FL_PBG_All[i] not in UW.columns:
            raise KeyError(f"Required field {FL_PBG_All[i]} is missing from data. ")
    print(f"1 --- UW imported.")

    # loan type 1, 2 only, 
    UW = UW[UW[FL_target].isin([1, 2]) & UW["ACTION"].isin([1, 2])]
    dist1 = UW[FL_target].fillna(-1).value_counts().reset_index()
    print(dist1)

    # FL_target
    UW[FL_target] = UW[FL_target].map({1:0, 2:1})


    # ===== Target, Flag distribution =====
    crosstab_out = {}
    for flag in FL_PBG_All:
        crosstab = pd.crosstab(UW[flag], UW[FL_target], margins=True)
        crosstab_out[flag] = crosstab
        print(f"crosstab is: {crosstab}")
    crosstab_df = pd.concat(crosstab_out.values(), 
                            keys=crosstab_out.keys(), names=["PBG", FL_target])
    print(f"2 --- UW flags and target crosstab_df created.")
    #print(crosstab_df)

    # ===== Sig. testing == RAW ================
    sig_results = []
    for i in [FL_PBG_RaceEth, FL_PBG_Gen, FL_PBG_Age]:
        for flag in i[1:]:
            _df = UW[(UW[i[0]]==1) | (UW[flag]==1)][[FL_target, flag]]
            X = _df[flag]
            y = _df[FL_target]
            X = sm.add_constant(X) # intercept term
            model = sm.Logit(y, X).fit( method='lbfgs', disp=0) # penalty='none',no convergence messages, 'bfgs' important
            #model = LogisticRegression(fit_intercept=True, solver='lbfgs', penalty=None, tol=0.00000001)
            #model.fit(X,y)
            estimates = model.params
            p_values = model.pvalues
            odds_ratio_pbg = str(round(np.exp(estimates[flag]), 3))
            # results
            sig_results.append({
                "PBG": flag,
                "Intercept": estimates['const'],
                f"Estiamte": estimates[flag],
                f"P-Value": p_values[flag],
                "Odds_Ratio_PBG": odds_ratio_pbg,
                "Significance": "Significant" if p_values[flag] < 0.05 else "Not Significant"
            }) 
    sig_results_df = pd.DataFrame(sig_results)


    # ===== Sig. testing == RAW ================
    modeled_sig_results = []
    for j in [FL_PBG_RaceEth, FL_PBG_Gen, FL_PBG_Age]:
        for flag in j[1:]:
            _df = UW[(UW[j[0]]==1) | (UW[flag]==1)][[FL_target, flag],X_features]
            X = _df[[flag],X_features]
            y = _df[FL_target]
            X = sm.add_constant(X) # intercept term
            model = sm.Logit(y, X).fit( method='lbfgs', disp=0) # penalty='none',no convergence messages, 'bfgs' important
            #model = LogisticRegression(fit_intercept=True, solver='lbfgs', penalty=None, tol=0.00000001)
            #model.fit(X,y)
            estimates = model.params
            p_values = model.pvalues
            odds_ratio_pbg = str(round(np.exp(estimates[flag]), 3))
            # results
            modeled_sig_results.append({
                "PBG": flag,
                "Intercept": estimates['const'],
                f"Estiamte": estimates[flag],
                f"P-Value": p_values[flag],
                "Odds_Ratio_PBG": odds_ratio_pbg,
                "Significance": "Significant" if p_values[flag] < 0.05 else "Not Significant"
            }) 
    modeled_sig_results_df = pd.DataFrame(modeled_sig_results)

    # # ===== export ==================
    outputfile = os.path.join(Out_path, "UW_PBG_RAW_Tests.xlsx")
    with pd.ExcelWriter(outputfile, engine="openpyxl") as writer:
        crosstab_df.to_excel(writer, sheet_name="Distributions")
        sig_results_df.to_excel(writer, sheet_name="Raw Sig. Tests", index=False)
        modeled_sig_results.to_excel(writer, sheet_name="Raw Sig. Tests", index=False)

except Exception as e:
    print(f"An error has occured: {e}")
