import streamlit as st 
import pandas as pd 
import plotly.graph_objects as go

# Title of the app
st.title('Interactive Data Visualization')

# Sidebar for navigation (only file upload and main navigation)
st.sidebar.header('Navigation')

# Upload CSV or Excel File
uploaded_file = st.sidebar.file_uploader("Choose a CSV or Excel file", type=["csv", "xlsx"]) 

if uploaded_file is not None: 
    # Determine the file type and read the file into a DataFrame accordingly 
    if uploaded_file.name.endswith('.csv'): 
        data = pd.read_csv(uploaded_file) 
    elif uploaded_file.name.endswith('.xlsx'): 
        data = pd.read_excel(uploaded_file)

    # Check if essential columns are present
    if 'BANK' in data.columns and all(col in data.columns for col in ['White_Non_Hispanic_YN', 'Black_Final_YN', 'Asian_Final_YN']):
        
        # Tab selection in sidebar
        tab_selection = st.sidebar.radio('Select Report Type', 
                                       options=['Interactive Charts', 'Appraisal Report'])
        
        if tab_selection == 'Interactive Charts':
            # Create two columns for main content and filters
            col1, col2 = st.columns([3, 1])  # 3:1 ratio for content:filters
            
            with col2:
                st.subheader('Filters')
                
                # Chart type selection
                chart_type = st.selectbox('Select Chart Type', options=['Data Table', 'Bar Chart', 'Line Chart'])
                
                # Filtering Options
                actions = data['ACTION'].fillna("Missing").unique()
                lien_statuses = data['LIEN_STATUS'].fillna("Missing").unique()
                loan_types = data['LOANTYPE'].fillna("Missing").unique()
                purposes = data['PURPOSE'].fillna("Missing").unique()
                occupancy_types = data['OCCUPANCYTYPE'].fillna("Missing").unique()
                
                selected_actions = st.multiselect('Select ACTION', options=actions, default=list(actions))
                selected_lien_statuses = st.multiselect('Select LIEN STATUS', options=lien_statuses, default=list(lien_statuses))
                selected_loan_types = st.multiselect('Select LOANTYPE', options=loan_types, default=list(loan_types))
                selected_purposes = st.multiselect('Select PURPOSE', options=purposes, default=list(purposes))
                selected_occupancy_types = st.multiselect('Select OCCUPANCYTYPE', options=occupancy_types, default=list(occupancy_types))

            # Filtering the data based on selections
            filtered_data = data[
                (data['ACTION'].fillna("Missing").isin(selected_actions)) &
                (data['LIEN_STATUS'].fillna("Missing").isin(selected_lien_statuses)) &
                (data['LOANTYPE'].fillna("Missing").isin(selected_loan_types)) &
                (data['PURPOSE'].fillna("Missing").isin(selected_purposes)) &
                (data['OCCUPANCYTYPE'].fillna("Missing").isin(selected_occupancy_types))
            ]

            with col1:
                # If no data remains after filtering
                if filtered_data.empty:
                    st.warning("No data available for the selected filters. Please adjust your selections.")
                else:
                    # Grouping the filtered data based on 'BANK' and counting
                    count_data = filtered_data.groupby('BANK').agg({
                        'White_Non_Hispanic_YN': 'sum', 
                        'Black_Final_YN': 'sum', 
                        'Asian_Final_YN': 'sum'
                    }).reset_index()
                    
                    # Add total count column
                    total_counts = filtered_data.groupby('BANK').size().reset_index(name='ALL')
                    count_data = count_data.merge(total_counts, on='BANK')

                    # Display content based on chart type selection
                    if chart_type == 'Data Table':
                        st.subheader('Data Table')
                        st.dataframe(count_data, use_container_width=True)
                        
                    elif chart_type == 'Bar Chart':
                        st.subheader('Bar Chart')
                        
                        # Select which bank to display
                        selected_bank = st.selectbox('Select BANK for Bar Chart', 
                                                   options=count_data['BANK'].tolist())
                        
                        # Filter data for selected bank
                        bank_data = count_data[count_data['BANK'] == selected_bank]
                        
                        # Create bar chart with ALL (total count) only
                        fig = go.Figure(data=[go.Bar(
                            x=[selected_bank], 
                            y=bank_data['ALL']
                        )])
                        fig.update_layout(
                            title=f'Bar Chart: Total Record Count for {selected_bank}',
                            xaxis_title='BANK',
                            yaxis_title='Total Count of Records'
                        )
                        st.plotly_chart(fig, use_container_width=True)

                    elif chart_type == 'Line Chart':
                        st.subheader('Line Chart')
                        
                        # Create loan type counts for line chart
                        loan_type_counts = filtered_data.groupby('LOANTYPE').agg({
                            'White_Non_Hispanic_YN': 'sum', 
                            'Black_Final_YN': 'sum', 
                            'Asian_Final_YN': 'sum'
                        }).reset_index()
                        
                        y_col = st.selectbox('Select Count Column for Line Chart', 
                                           options=['White_Non_Hispanic_YN', 'Black_Final_YN', 'Asian_Final_YN'])

                        fig = go.Figure(data=[go.Scatter(
                            x=loan_type_counts['LOANTYPE'], 
                            y=loan_type_counts[y_col], 
                            mode='lines+markers'
                        )])
                        fig.update_layout(
                            title=f'Line Chart: {y_col} counts by LOANTYPE',
                            xaxis_title='LOANTYPE',
                            yaxis_title=f'Count of {y_col}'
                        )
                        st.plotly_chart(fig, use_container_width=True)
        
        elif tab_selection == 'Appraisal Report':
            st.subheader('Appraisal Report')
            
            # Show the data table without filters
            count_data = data.groupby('ACTION').agg({
                'White_Non_Hispanic_YN': 'sum', 
                'Black_Final_YN': 'sum', 
                'Hispanic_Final': 'sum',
                'Asian_Final_YN': 'sum',
                'AmercInd_Final_YN': 'sum',
                'PacIsland_Final_YN':'sum',
                'Gender_Control_YN':'sum',
                'Gender_Target_YN':'sum',
                'Age_Control_YN':'sum',
                'Age_Target_YN':'sum',
                'LM_YN':'sum',
                'MM_YN':'sum',
                'MBH_YN':'sum',
                'HM_YN':'sum'
            }).reset_index()
            
            # Add total count column
            total_counts = data.groupby('ACTION').size().reset_index(name='ALL')
            count_data = count_data.merge(total_counts, on='ACTION')
            
            # Display the data table
            st.dataframe(count_data, use_container_width=True)
            
            # Add some summary statistics
            st.subheader('Summary Statistics')
            col1, col2, col3, col4 = st.columns(4)
            
            with col1:
                st.metric("Total ACTION", len(count_data))
            with col2:
                st.metric("Total Records", count_data['ALL'].sum())
            with col3:
                st.metric("White Non-Hispanic Total", count_data['White_Non_Hispanic_YN'].sum())
            with col4:
                st.metric("Total Minority Records", 
                         count_data['Black_Final_YN'].sum() + count_data['Asian_Final_YN'].sum())
    
    else:
        st.warning("The file must contain the 'ACTION', 'White_Non_Hispanic_YN', 'Black_Final_YN', and 'Asian_Final_YN' columns.")
        st.info("Available columns: " + ", ".join(data.columns.tolist()))

else: 
    st.info("Please upload a CSV or Excel file to get started.")
