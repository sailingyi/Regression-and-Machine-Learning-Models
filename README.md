
import streamlit as st
import pandas as pd
import plotly.graph_objects as go

# Title of the app
st.title('Interactive Data Visualization')

# Sidebar for navigation
st.sidebar.header('Navigation')
# Upload CSV or Excel File
uploaded_file = st.sidebar.file_uploader("Choose a CSV or Excel file", type=["csv", "xlsx"])
chart_type = st.sidebar.selectbox('Select Chart Type', options=['Data Table', 'Bar Chart', 'Line Chart'])

if uploaded_file is not None:
    # Determine the file type and read the file into a DataFrame accordingly
    if uploaded_file.name.endswith('.csv'):
        data = pd.read_csv(uploaded_file)
    elif uploaded_file.name.endswith('.xlsx'):
        data = pd.read_excel(uploaded_file)

    # Check if essential columns are present
    if 'BANK' in data.columns and all(col in data.columns for col in ['White_Non_Hispanic_YN', 'Black_Final_YN', 'Asian_Final_YN']):
        
        # Filtering Options
        actions = data['ACTION'].fillna("Mising").unique()
        lien_statuses = data['LIEN_STATUS'].fillna("Mising").unique()
        loan_types = data['LOANTYPE'].fillna("Mising").unique()
        purposes = data['PURPOSE'].fillna("Mising").unique()
        occupancy_types = data['OCCUPANCYTYPE'].fillna("Mising").unique()
        
        selected_actions = st.sidebar.multiselect('Select ACTION', options=actions, default=list(actions))
        selected_lien_statuses = st.sidebar.multiselect('Select LIEN STATUS', options=lien_statuses, default=list(lien_statuses))
        selected_loan_types = st.sidebar.multiselect('Select LOANTYPE', options=loan_types, default=list(loan_types))
        selected_purposes = st.sidebar.multiselect('Select PURPOSE', options=purposes, default=list(purposes))
        selected_occupancy_types = st.sidebar.multiselect('Select OCCUPANCYTYPE', options=occupancy_types, default=list(occupancy_types))

        # Filtering the data based on selections
        filtered_data = data[
            (data['ACTION'].isin(selected_actions)) &
            (data['LIEN_STATUS'].isin(selected_lien_statuses)) &
            (data['LOANTYPE'].isin(selected_loan_types)) &
            (data['PURPOSE'].isin(selected_purposes)) &
            (data['OCCUPANCYTYPE'].isin(selected_occupancy_types))
        ]

        # If no data remains after filtering
        if filtered_data.empty:
            st.warning("No data available for the selected filters. Please adjust your selections.")
        else:
            # Grouping the filtered data based on 'BANK' and counting
            count_data = filtered_data.groupby('BANK').agg({'White_Non_Hispanic_YN': 'sum', 'Black_Final_YN': 'sum', 'Asian_Final_YN': 'sum'}).reset_index()

            # Tabs for different chart types
            tab1, tab2 = st.tabs(["Data Table", "Charts"])

            # Create a chart based on selected type
            with tab1:
                # Display the DataFrame in an interactive table
                st.subheader('Data Table')
                st.dataframe(count_data)

            with tab2:
                chart_option = st.selectbox('Select Chart Type', options=[
                    'Bar Chart', 'Line Chart'
                ])
            
                if chart_type == 'Bar Chart':
                    st.subheader('Bar Chart')
                    x_col = st.selectbox('Select BANK column for Bar Chart', options=count_data['BANK'])
                    y_col = st.selectbox('Select Count Column', options=['White_Non_Hispanic_YN', 'Black_Final_YN', 'Asian_Final_YN'])

                    fig = go.Figure(data=[go.Bar(x=count_data['BANK'], y=count_data[y_col])])
                    fig.update_layout(title=f'Bar Chart: {y_col} counts by BANK',
                                    xaxis_title='BANK',
                                    yaxis_title=f'Count of {y_col}')

                    # Display the bar chart
                    st.plotly_chart(fig)

                elif chart_type == 'Line Chart':
                    st.subheader('Line Chart')
                    loan_type_counts = filtered_data.groupby('LOANTYPE').agg({'White_Non_Hispanic_YN': 'sum', 
                                                                             'Black_Final_YN': 'sum', 
                                                                             'Asian_Final_YN': 'sum'}).reset_index()
                    x_col = st.selectbox('Select LOANTYPE column for Bar Chart', options=loan_type_counts['LOANTYPE'])
                    y_col = st.selectbox('Select Count Column for Line Chart', options=['White_Non_Hispanic_YN', 'Black_Final_YN', 'Asian_Final_YN'], key='line_y')

                    fig = go.Figure(data=[go.Scatter(x=count_data['LOANTYPE'], y=count_data[y_col], mode='lines+markers')])
                    fig.update_layout(title=f'Line Chart: {y_col} counts by LOANTYPE',
                                xaxis_title='LOANTYPE',
                                yaxis_title=f'Count of {y_col}')
                else:
                        st.warning("The file must contain the 'LOANTYPE', 'BANK', 'White_Non_Hispanic_YN', 'Black_Final_YN', and 'Asian_Final_YN' columns.")

else:
    st.info("Please upload a CSV or Excel file to get started.")

