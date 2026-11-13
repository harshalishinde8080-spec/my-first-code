import streamlit as st
import pandas as pd
import plotly.express as px
import os

st.set_page_config(
    page_title="Student Social Media & Mental Health Analytics",
    page_icon="📊",
    layout="wide"
)

st.title("📱 Student Social Media Usage & Mental Health Dashboard")
st.markdown("**Interactive Analytics Dashboard to Explore Social Media Habits, Sleep Patterns, and Mental Well-being.**")
st.markdown("---")

@st.cache_data
def load_data():
    possible_files = ["mental health2.csv.xlsx", "mental health2.csv", "Student Social Media And Mental Health Impact.xlsx"]
    file_found = None
    for f in possible_files:
        if os.path.exists(f):
            file_found = f
            break
    if not file_found:
        st.error("⚠️ File nahi mili! Kripya check karein ki file properly uploaded hai.")
        return None
    try:
        if file_found.endswith('.xlsx'):
            df = pd.read_excel(file_found)
        else:
            df = pd.read_csv(file_found)
        return df
    except Exception as e:
        st.error(f"Error loading file: {e}")
        return None

df = load_data()

if df is not None:
    st.sidebar.header("🔍 Controls & Filters")
    search_query = st.sidebar.text_input("🔎 Search (Country / Platform / Purpose):", "")
    st.sidebar.markdown("---")
    
    platform_col = "Most_Platform" if "Most_Platform" in df.columns else ("Most_Used_Platform" if "Most_Used_Platform" in df.columns else None)
    
    platforms = sorted(df[platform_col].dropna().unique()) if platform_col and platform_col in df.columns else []
    stress_levels = sorted(df["Stress_Level"].dropna().unique()) if "Stress_Level" in df.columns else []
    academic_levels = sorted(df["Academic_Level"].dropna().unique()) if "Academic_Level" in df.columns else []
    
    selected_platforms = st.sidebar.multiselect("Filter Platforms:", options=platforms, default=platforms)
    selected_stress = st.sidebar.multiselect("Filter Stress Levels:", options=stress_levels, default=stress_levels)
    selected_academic = st.sidebar.multiselect("Filter Academic Level:", options=academic_levels, default=academic_levels)
    
    filtered_df = df.copy()
    if platform_col and selected_platforms:
        filtered_df = filtered_df[filtered_df[platform_col].isin(selected_platforms)]
    if "Stress_Level" in filtered_df.columns and selected_stress:
        filtered_df = filtered_df[filtered_df["Stress_Level"].isin(selected_stress)]
    if "Academic_Level" in filtered_df.columns and selected_academic:
        filtered_df = filtered_df[filtered_df["Academic_Level"].isin(selected_academic)]
        
    if search_query:
        query = search_query.lower()
        filtered_df = filtered_df[
            filtered_df.astype(str).apply(lambda row: row.str.lower().str.contains(query).any(), axis=1)
        ]
        
    if filtered_df.empty:
        st.warning("⚠️ No data matches your search or filter criteria.")
    else:
        col1, col2, col3, col4, col5 = st.columns(5)
        col1.metric("Total Records", f"{len(filtered_df):,}")
        col2.metric("Avg Daily Usage", f"{filtered_df['Avg_Daily_Usage_Hours'].mean():.2f} hrs" if "Avg_Daily_Usage_Hours" in filtered_df.columns else "N/A")
        col3.metric("Avg Mental Score", f"{filtered_df['Mental_Health_Score'].mean():.2f} / 10" if "Mental_Health_Score" in filtered_df.columns else "N/A")
        col4.metric("Avg Sleep Hours", f"{filtered_df['Sleep_Hours_Per_Night'].mean():.2f} hrs" if "Sleep_Hours_Per_Night" in filtered_df.columns else "N/A")
        col5.metric("Avg Study Hours", f"{filtered_df['Study_Hours'].mean():.2f} hrs" if "Study_Hours" in filtered_df.columns else "N/A")
        
        st.markdown("---")
        row1_col1, row1_col2 = st.columns(2)
        
        with row1_col1:
            st.subheader("📈 Usage Hours vs Mental Health Score")
            if all(col in filtered_df.columns for col in ["Avg_Daily_Usage_Hours", "Mental_Health_Score"]):
                fig_scatter = px.scatter(
                    filtered_df, x="Avg_Daily_Usage_Hours", y="Mental_Health_Score",
                    color=platform_col if platform_col else None,
                    size="Daily_Unlocks" if "Daily_Unlocks" in filtered_df.columns else None,
                    hover_data=[c for c in ["Age", "Stress_Level", "Academic_Level"] if c in filtered_df.columns],
                    title="Correlation: Screen Time & Mental Wellness"
                )
                st.plotly_chart(fig_scatter, use_container_width=True)
            else:
                st.info("Required columns for scatter plot not found.")
                
        with row1_col2:
            st.subheader("📊 Platform Popularity Breakdown")
            if platform_col and platform_col in filtered_df.columns:
                platform_counts = filtered_df[platform_col].value_counts().reset_index()
                platform_counts.columns = ['Platform', 'Count']
                fig_bar = px.bar(platform_counts, x="Platform", y="Count", color="Platform", title="Student Distribution Across Platforms", text="Count")
                st.plotly_chart(fig_bar, use_container_width=True)
            else:
                st.info("Platform column not found.")
                
        row2_col1, row2_col2 = st.columns(2)
        
        with row2_col1:
            st.subheader("💤 Stress Levels vs Sleep Quality")
            if all(col in filtered_df.columns for col in ["Stress_Level", "Sleep_Hours_Per_Night"]):
                fig_box = px.box(filtered_df, x="Stress_Level", y="Sleep_Hours_Per_Night", color="Stress_Level", title="Sleep Hours Distribution by Stress Category")
                st.plotly_chart(fig_box, use_container_width=True)
            else:
                st.info("Required columns not found.")
                
        with row2_col2:
            st.subheader("🎯 Primary Purpose of Social Media")
            if "Purpose_Of_Use" in filtered_df.columns:
                fig_pie = px.pie(filtered_df, names="Purpose_Of_Use", hole=0.4, title="Usage Purpose Share")
                st.plotly_chart(fig_pie, use_container_width=True)
            else:
                st.info("Purpose_Of_Use column not found.")
                
        st.markdown("---")
        st.subheader("📋 Detailed Data Records Explorer")
        st.dataframe(filtered_df, use_container_width=True)
        
        csv_data = filtered_df.to_csv(index=False).encode('utf-8')
        st.download_button(
            label="📥 Download Filtered Dataset as CSV",
            data=csv_data,
            file_name="social_media_mental_health_filtered.csv",
            mime="text/csv",
        )