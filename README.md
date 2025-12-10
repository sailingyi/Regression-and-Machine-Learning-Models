import pandas as pd
import numpy as np
import warnings

#### Import Data ####

convfolder = "Z:/Shared/Compliance/Fair & Responsible Banking/Fair Lending Reviews/24FY WM Underwriting Reviews/Conventional/Data"
convfile = "24FY WM Conv UW - Input Data.csv"

fhausdafolder = "Z:/Shared/Compliance/Fair & Responsible Banking/Fair Lending Reviews/24FY WM Underwriting Reviews/FHA_USDA/Data"
fhausdafile = "24FY WM FHA_USDA UW - Input Data.csv"

print("Loading data files...")

# Import conv HMDA (action codes 1,2,3)
uw_conv = pd.read_csv(f"{convfolder}/{convfile}")
print(f"Loaded conventional data: {uw_conv.shape[0]} rows, {uw_conv.shape[1]} columns")

# Convert CENTROID to string to match FHA
uw_conv['CENTROID'] = uw_conv['CENTROID'].astype(str)

# Import fha/usda HMDA (action codes 1,2,3)
uw_fhausda = pd.read_csv(f"{fhausdafolder}/{fhausdafile}")
print(f"Loaded FHA/USDA data: {uw_fhausda.shape[0]} rows, {uw_fhausda.shape[1]} columns")

# Filter to FHA applications only
uw_fha = uw_fhausda[uw_fhausda['LOANTYPE'] == "2"].copy()
print(f"Filtered to FHA only: {uw_fha.shape[0]} rows")

# Merge conv and fha together
conv_fha = pd.concat([uw_conv, uw_fha], ignore_index=True)
print(f"Combined dataset: {conv_fha.shape[0]} rows, {conv_fha.shape[1]} columns")

#### Data Type Validation and Cleaning ####

print("\n" + "="*60)
print("VALIDATING AND CLEANING DATA TYPES")
print("="*60)

# Define expected numeric fields
numeric_fields = [
    'DTIRATIO', 'CLTV', 'CREDITSCORE', 'LOANAMOUNTINDOLLARS', 
    'Conf_LoanLimit', 'TOTALUNITS', 'PURPOSE', 'OCCUPANCYTYPE', 
    'ACTION'
]

# Define expected categorical/text fields
text_fields = ['LOANTYPE', 'Property_Group', 'CENTROID']

# Function to safely convert to numeric
def safe_numeric_convert(series, field_name):
    """Convert series to numeric, with error reporting"""
    original_dtype = series.dtype
    
    # Try to convert to numeric
    converted = pd.to_numeric(series, errors='coerce')
    
    # Count how many values were coerced to NaN (were non-numeric)
    non_numeric_count = converted.isna().sum() - series.isna().sum()
    
    if non_numeric_count > 0:
        print(f"⚠️  {field_name}: Found {non_numeric_count} non-numeric values, converted to NaN")
        print(f"   Original dtype: {original_dtype} → Converting to numeric")
    
    return converted

# Function to safely convert to string
def safe_text_convert(series, field_name):
    """Convert series to string, with error reporting"""
    original_dtype = series.dtype
    
    if original_dtype not in ['object', 'string']:
        print(f"⚠️  {field_name}: Converting from {original_dtype} to string")
    
    # Convert to string, replacing NaN with empty string
    converted = series.astype(str).replace('nan', '')
    
    return converted

# Validate and convert numeric fields
print("\nValidating numeric fields:")
for field in numeric_fields:
    if field in conv_fha.columns:
        conv_fha[field] = safe_numeric_convert(conv_fha[field], field)
    else:
        print(f"❌ WARNING: Expected field '{field}' not found in data!")
        # Create the field with NaN values
        conv_fha[field] = np.nan

# Validate and convert text fields
print("\nValidating text/categorical fields:")
for field in text_fields:
    if field in conv_fha.columns:
        conv_fha[field] = safe_text_convert(conv_fha[field], field)
    else:
        print(f"❌ WARNING: Expected field '{field}' not found in data!")
        # Create the field with empty strings
        conv_fha[field] = ''

# Fill missing values with appropriate defaults
print("\n" + "="*60)
print("FILLING MISSING VALUES")
print("="*60)

fillna_config = {
    'DTIRATIO': -999,  # Use -999 to indicate missing, will be caught in logic
    'CLTV': -999,
    'CREDITSCORE': -999,
    'LOANAMOUNTINDOLLARS': 0,
    'Conf_LoanLimit': 0,
    'TOTALUNITS': 1,  # Default to 1 unit if missing
    'PURPOSE': -999,
    'OCCUPANCYTYPE': -999,
    'ACTION': -999,
    'Property_Group': 'Unknown',
    'LOANTYPE': 'Unknown'
}

for field, fill_value in fillna_config.items():
    if field in conv_fha.columns:
        null_count = conv_fha[field].isna().sum()
        if null_count > 0:
            print(f"  {field}: Filling {null_count} missing values with {fill_value}")
            conv_fha[field].fillna(fill_value, inplace=True)

print("\nData validation complete!")
print("="*60 + "\n")

#### Creating Regression Variables ##########

print("Creating regression variables...")

conv_fha_regvars = conv_fha.copy()

# DTI Vars
conv_fha_regvars['DTI_LessEq43'] = np.where(
    (conv_fha_regvars['DTIRATIO'].isna()) | (conv_fha_regvars['DTIRATIO'] == -999), 0,
    np.where((conv_fha_regvars['DTIRATIO'] <= 43) & (conv_fha_regvars['DTIRATIO'] >= 0), 1, 0)
)

conv_fha_regvars['DTI_Great43_LessEq50'] = np.where(
    (conv_fha_regvars['DTIRATIO'].isna()) | (conv_fha_regvars['DTIRATIO'] == -999), 0,
    np.where((conv_fha_regvars['DTIRATIO'] > 43) & (conv_fha_regvars['DTIRATIO'] <= 50), 1, 0)
)

conv_fha_regvars['DTI_Great50_orNegative_orMissing'] = np.where(
    (conv_fha_regvars['DTIRATIO'] > 50) | 
    (conv_fha_regvars['DTIRATIO'] < 0) | 
    (conv_fha_regvars['DTIRATIO'] == -999) |
    conv_fha_regvars['DTIRATIO'].isna(), 1, 0
)

# CLTV Vars
conv_fha_regvars['CLTV_LessEq75'] = np.where(
    (conv_fha_regvars['CLTV'].isna()) | (conv_fha_regvars['CLTV'] == -999), 0,
    np.where(conv_fha_regvars['CLTV'] <= 75, 1, 0)
)

conv_fha_regvars['CLTV_Great75_LessEq80'] = np.where(
    (conv_fha_regvars['CLTV'].isna()) | (conv_fha_regvars['CLTV'] == -999), 0,
    np.where((conv_fha_regvars['CLTV'] > 75) & (conv_fha_regvars['CLTV'] <= 80), 1, 0)
)

conv_fha_regvars['CLTV_Great80_LessEq95'] = np.where(
    (conv_fha_regvars['CLTV'].isna()) | (conv_fha_regvars['CLTV'] == -999), 0,
    np.where((conv_fha_regvars['CLTV'] > 80) & (conv_fha_regvars['CLTV'] <= 95), 1, 0)
)

conv_fha_regvars['CLTV_Great95_LessEq100'] = np.where(
    (conv_fha_regvars['CLTV'].isna()) | (conv_fha_regvars['CLTV'] == -999), 0,
    np.where((conv_fha_regvars['CLTV'] > 95) & (conv_fha_regvars['CLTV'] <= 100), 1, 0)
)

conv_fha_regvars['CLTV_Great100_LessEq105'] = np.where(
    (conv_fha_regvars['CLTV'].isna()) | (conv_fha_regvars['CLTV'] == -999), 0,
    np.where((conv_fha_regvars['CLTV'] > 100) & (conv_fha_regvars['CLTV'] <= 105), 1, 0)
)

conv_fha_regvars['CLTV_Great105'] = np.where(
    (conv_fha_regvars['CLTV'].isna()) | (conv_fha_regvars['CLTV'] == -999), 0,
    np.where(conv_fha_regvars['CLTV'] > 105, 1, 0)
)

conv_fha_regvars['CLTV_Missing'] = np.where(
    (conv_fha_regvars['CLTV'].isna()) | (conv_fha_regvars['CLTV'] == -999), 1, 0
)

# Loan Purpose
conv_fha_regvars['Purchase'] = np.where(conv_fha_regvars['PURPOSE'] == 1, 1, 0)
conv_fha_regvars['RT_Refinance'] = np.where(conv_fha_regvars['PURPOSE'] == 31, 1, 0)
conv_fha_regvars['CashOut_Refinance'] = np.where(
    conv_fha_regvars['PURPOSE'].isin([2, 32, 4]), 1, 0
)

# Occupancy
conv_fha_regvars['OwnerOccupied'] = np.where(conv_fha_regvars['OCCUPANCYTYPE'] == 1, 1, 0)
conv_fha_regvars['Second_Home'] = np.where(conv_fha_regvars['OCCUPANCYTYPE'] == 2, 1, 0)
conv_fha_regvars['Investment_Property'] = np.where(conv_fha_regvars['OCCUPANCYTYPE'] == 3, 1, 0)

# Property
conv_fha_regvars['Trad_1to4'] = np.where(conv_fha_regvars['Property_Group'] == "Trad 1-4", 1, 0)
conv_fha_regvars['Manufactured_Home'] = np.where(conv_fha_regvars['Property_Group'] == "Manufactured", 1, 0)
conv_fha_regvars['Condo'] = np.where(conv_fha_regvars['Property_Group'] == "Condo", 1, 0)
conv_fha_regvars['Other_Property'] = np.where(
    (conv_fha_regvars['Property_Group'].isna()) | (conv_fha_regvars['Property_Group'] == '') | (conv_fha_regvars['Property_Group'] == 'Unknown'), 1,
    np.where(
        (conv_fha_regvars['Trad_1to4'] == 0) & 
        (conv_fha_regvars['Manufactured_Home'] == 0) & 
        (conv_fha_regvars['Condo'] == 0), 1, 0
    )
)

# Credit Score
conv_fha_regvars['Cred_GreatEq300_Less620'] = np.where(
    (conv_fha_regvars['CREDITSCORE'].isna()) | (conv_fha_regvars['CREDITSCORE'] == -999), 0,
    np.where((conv_fha_regvars['CREDITSCORE'] >= 300) & (conv_fha_regvars['CREDITSCORE'] < 620), 1, 0)
)

conv_fha_regvars['Cred_GreatEq620_Less640'] = np.where(
    (conv_fha_regvars['CREDITSCORE'].isna()) | (conv_fha_regvars['CREDITSCORE'] == -999), 0,
    np.where((conv_fha_regvars['CREDITSCORE'] >= 620) & (conv_fha_regvars['CREDITSCORE'] < 640), 1, 0)
)

conv_fha_regvars['Cred_GreatEq640_Less680'] = np.where(
    (conv_fha_regvars['CREDITSCORE'].isna()) | (conv_fha_regvars['CREDITSCORE'] == -999), 0,
    np.where((conv_fha_regvars['CREDITSCORE'] >= 640) & (conv_fha_regvars['CREDITSCORE'] < 680), 1, 0)
)

conv_fha_regvars['Cred_GreatEq680_Less700'] = np.where(
    (conv_fha_regvars['CREDITSCORE'].isna()) | (conv_fha_regvars['CREDITSCORE'] == -999), 0,
    np.where((conv_fha_regvars['CREDITSCORE'] >= 680) & (conv_fha_regvars['CREDITSCORE'] < 700), 1, 0)
)

conv_fha_regvars['Cred_GreatEq700_Less740'] = np.where(
    (conv_fha_regvars['CREDITSCORE'].isna()) | (conv_fha_regvars['CREDITSCORE'] == -999), 0,
    np.where((conv_fha_regvars['CREDITSCORE'] >= 700) & (conv_fha_regvars['CREDITSCORE'] < 740), 1, 0)
)

conv_fha_regvars['Cred_GreatEq740'] = np.where(
    (conv_fha_regvars['CREDITSCORE'].isna()) | (conv_fha_regvars['CREDITSCORE'] == -999), 0,
    np.where((conv_fha_regvars['CREDITSCORE'] >= 740) & (conv_fha_regvars['CREDITSCORE'] <= 850), 1, 0)
)

conv_fha_regvars['Cred_Missing'] = np.where(
    (conv_fha_regvars['CREDITSCORE'].isna()) | (conv_fha_regvars['CREDITSCORE'] == -999), 1, 0
)

conv_fha_regvars['Cred_NA'] = np.where(
    (conv_fha_regvars['CREDITSCORE'].isna()) | (conv_fha_regvars['CREDITSCORE'] == -999), 0,
    np.where(conv_fha_regvars['CREDITSCORE'] == 8888, 1, 0)
)

# LoanAmount
conv_fha_regvars['LoanAmount_Conforming'] = np.where(
    conv_fha_regvars['LOANAMOUNTINDOLLARS'] <= conv_fha_regvars['Conf_LoanLimit'], 1, 0
)

conv_fha_regvars['LoanAmount_GreatConforming'] = np.where(
    conv_fha_regvars['LOANAMOUNTINDOLLARS'] > conv_fha_regvars['Conf_LoanLimit'], 1, 0
)

# Total Units
conv_fha_regvars['One_Unit'] = np.where(conv_fha_regvars['TOTALUNITS'] == 1, 1, 0)
conv_fha_regvars['Multi_Unit'] = np.where(conv_fha_regvars['TOTALUNITS'] > 1, 1, 0)

# Adding Denial Indicator
conv_fha_regvars['Denied_YN'] = np.where(conv_fha_regvars['ACTION'] == 3, 1, 0)
conv_fha_regvars['Approved_YN'] = np.where(conv_fha_regvars['ACTION'] != 3, 1, 0)

print("Regression variables created successfully!")

#### Export Data ####

output_folder = "Z:/Shared/Compliance/Fair & Responsible Banking/Fair Lending Reviews/24FY WM Underwriting Reviews"
outputfile = "24FY WM ConvFHA UW.csv"

print(f"\nExporting data to: {output_folder}/{outputfile}")
conv_fha_regvars.to_csv(f"{output_folder}/{outputfile}", index=False)
print(f"✓ Export complete! Final dataset: {conv_fha_regvars.shape[0]} rows, {conv_fha_regvars.shape[1]} columns")
