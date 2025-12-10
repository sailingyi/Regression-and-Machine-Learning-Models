#### Load Packages and Functions ####

library('dplyr')

#### Import Data ####

convfolder <- "Z:/Shared/Compliance/Fair & Responsible Banking/Fair Lending Reviews/24FY WM Underwriting Reviews/Conventional/Data"
convfile <- "24FY WM Conv UW - Input Data.csv"

fhausdafolder <- "Z:/Shared/Compliance/Fair & Responsible Banking/Fair Lending Reviews/24FY WM Underwriting Reviews/FHA_USDA/Data"
fhausdafile <- "24FY WM FHA_USDA UW - Input Data.csv"


uw_conv <- read.csv(paste0(convfolder,"/",convfile))%>% # import conv HMDA (action codes 1,2,3)
                  dplyr::mutate(CENTROID = as.character(CENTROID)) # this field imported as numeric, while for FHA it imported as a character. Converting to match

uw_fhausda <- read.csv(paste0(fhausdafolder,"/",fhausdafile))# import fha/usda HMDA (action codes 1,2,3)

uw_fha <- uw_fhausda%>% # filter to FHA applications only
  dplyr::filter(LOANTYPE=="2")


conv_fha <- uw_conv%>% # merge conv and fha together
  dplyr::bind_rows(uw_fha)


#### Creating Regression Variables ##########

# code is copied from the Conv UW Regression (FY24) code file, used in the 24FY Conv UW model

conv_fha_regvars <- conv_fha%>%
  # DTI Vars
  dplyr::mutate(DTI_LessEq43 = ifelse(is.na(DTIRATIO),0, # omitted reference
                                      ifelse(DTIRATIO<=43 & DTIRATIO>=0,1,0)),
                DTI_Great43_LessEq50 = ifelse(is.na(DTIRATIO),0,
                                              ifelse(DTIRATIO>43 & DTIRATIO<=50,1,0)),
                DTI_Great50_orNegative_orMissing = ifelse(DTIRATIO>50|DTIRATIO<0|is.na(DTIRATIO),1,0))%>%
  # CLTV Vars
  dplyr::mutate(CLTV_LessEq75 = ifelse(is.na(CLTV),0,
                                       ifelse(CLTV<=75,1,0)),
                CLTV_Great75_LessEq80 = ifelse(is.na(CLTV),0, # omitted reference
                                               ifelse(CLTV >75 & CLTV<=80,1,0)),
                CLTV_Great80_LessEq95 = ifelse(is.na(CLTV),0,
                                               ifelse(CLTV>80 & CLTV<=95,1,0)),
                
                CLTV_Great95_LessEq100 = ifelse(is.na(CLTV),0,
                                                ifelse(CLTV >95 & CLTV<=100,1,0)),
                CLTV_Great100_LessEq105 = ifelse(is.na(CLTV),0,
                                                 ifelse(CLTV >100 & CLTV<=105,1,0)),
                CLTV_Great105 = ifelse(is.na(CLTV),0,
                                       ifelse(CLTV >105,1,0)),
                CLTV_Missing = ifelse(is.na(CLTV),1,0))%>%
  
  # Loan Purpose
  dplyr::mutate(Purchase = ifelse(PURPOSE ==1,1,0), # omitted reference
                RT_Refinance = ifelse(PURPOSE ==31,1,0),
                CashOut_Refinance = ifelse(PURPOSE %in% c(2,32,4),1,0))%>%
  
  # Occupancy
  dplyr::mutate(OwnerOccupied = ifelse(OCCUPANCYTYPE==1,1,0), # omitted reference
                Second_Home = ifelse(OCCUPANCYTYPE==2,1,0),
                Investment_Property = ifelse(OCCUPANCYTYPE==3,1,0))%>%
  
  # Property
  dplyr::mutate(Trad_1to4 = ifelse(Property_Group=="Trad 1-4",1,0), # omitted reference
                Manufactured_Home = ifelse(Property_Group=="Manufactured",1,0),
                Condo = ifelse(Property_Group=="Condo",1,0),
                Other_Property = ifelse(is.na(Property_Group),1,
                                        ifelse(Trad_1to4==0 & Manufactured_Home==0 &
                                                 Condo==0,1,0)))%>%
  
  # Credit Score
  dplyr::mutate(Cred_GreatEq300_Less620 = ifelse(is.na(CREDITSCORE),0,
                                                 ifelse(CREDITSCORE >=300 & CREDITSCORE<620,1,0)),
                Cred_GreatEq620_Less640 = ifelse(is.na(CREDITSCORE),0,
                                                 ifelse(CREDITSCORE >=620 & CREDITSCORE<640,1,0)),
                Cred_GreatEq640_Less680 = ifelse(is.na(CREDITSCORE),0,
                                                 ifelse(CREDITSCORE >=640 & CREDITSCORE <680,1,0)),
                Cred_GreatEq680_Less700 = ifelse(is.na(CREDITSCORE),0,
                                                 ifelse(CREDITSCORE >=680 & CREDITSCORE<700,1,0)),
                Cred_GreatEq700_Less740 = ifelse(is.na(CREDITSCORE),0,
                                                 ifelse(CREDITSCORE >=700 & CREDITSCORE <740,1,0)),
                Cred_GreatEq740 = ifelse(is.na(CREDITSCORE),0, # omitted reference
                                         ifelse(CREDITSCORE >=740 & CREDITSCORE <=850,1,0)),
                
                Cred_Missing = ifelse(is.na(CREDITSCORE),1,0),
                Cred_NA = ifelse(is.na(CREDITSCORE),0,
                                 ifelse(CREDITSCORE==8888,1,0)))%>%
  
  # LoanAmount
  dplyr::mutate(LoanAmount_Conforming =  ifelse(LOANAMOUNTINDOLLARS <= Conf_LoanLimit,1,0), # omitted reference
                LoanAmount_GreatConforming = ifelse(LOANAMOUNTINDOLLARS > Conf_LoanLimit,1,0))%>%
  
  # Total Units
  dplyr::mutate(One_Unit = ifelse(TOTALUNITS==1,1,0), # omitted reference
                Multi_Unit = ifelse(TOTALUNITS>1,1,0))%>%
  
  # Adding Denial Indicator
  dplyr::mutate(Denied_YN = ifelse(ACTION==3,1,0),
                Approved_YN = ifelse(ACTION!=3,1,0))


#### Export Data ####

output_folder <-"Z:/Shared/Compliance/Fair & Responsible Banking/Fair Lending Reviews/24FY WM Underwriting Reviews"
outputfile <- "24FY WM ConvFHA UW.csv"

write.csv(conv_fha_regvars,file = paste0(output_folder,"/",outputfile),row.names = F)
