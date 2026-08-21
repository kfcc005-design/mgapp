import streamlit as st

import pandas as pd

import plotly.express as px

# 1. 페이지 설정

st.set_page_config(

    page_title="업무 지원 요청 대시보드",

    page_icon="📊",

    layout="wide"

)

# 커스텀 스타일 (폰트 및 여백)

st.markdown("""

    <style>

    .main-title {

        font-size: 28px;

        font-weight: 700;

        margin-bottom: 20px;

    }

    .metric-card {

        background-color: #f8f9fa;

        padding: 15px;

        border-radius: 10px;

        border: 1px solid #e9ecef;

    }

    </style>

""", unsafe_allow_html=True)

st.markdown('<div class="main-title">📊 업무 지원 요청 데이터 시각화 대시보드</div>', unsafe_allow_html=True)

# 2. 사이드바 - 파일 업로드 및 필터링

st.sidebar.header("📁 데이터 업로드")

uploaded_file = st.sidebar.file_uploader("CSV 파일을 업로드하세요", type=["csv"])

if uploaded_file is not None:

    # 데이터 로드

    df = pd.read_csv(uploaded_file)

    

    # 날짜 컬럼 변환

    if 'request_date' in df.columns:

        df['request_date'] = pd.to_datetime(df['request_date'])

    # 사이드바 필터 설정

    st.sidebar.markdown("---")

    st.sidebar.header("🔍 데이터 필터")

    categories = ['전체'] + list(df['category'].unique()) if 'category' in df.columns else ['전체']

    selected_category = st.sidebar.selectbox("요청 유형(Category)", categories)

    statuses = ['전체'] + list(df['status'].unique()) if 'status' in df.columns else ['전체']

    selected_status = st.sidebar.selectbox("처리 상태(Status)", statuses)

    urgencies = ['전체'] + list(df['urgency'].unique()) if 'urgency' in df.columns else ['전체']

    selected_urgency = st.sidebar.selectbox("긴급도(Urgency)", urgencies)

    # 필터 적용

    filtered_df = df.copy()

    if selected_category != '전체':

        filtered_df = filtered_df[filtered_df['category'] == selected_category]

    if selected_status != '전체':

        filtered_df = filtered_df[filtered_df['status'] == selected_status]

    if selected_urgency != '전체':

        filtered_df = filtered_df[filtered_df['urgency'] == selected_urgency]

    # 3. 주요 KPI 메트릭 표시

    col1, col2, col3, col4 = st.columns(4)

    with col1:

        st.metric("총 요청 건수", f"{len(filtered_df):,}건")

    with col2:

        completed_cnt = len(filtered_df[filtered_df['status'] == '완료']) if 'status' in filtered_df.columns else 0

        st.metric("완료 건수", f"{completed_cnt}건")

    with col3:

        high_urgency_cnt = len(filtered_df[filtered_df['urgency'] == '상']) if 'urgency' in filtered_df.columns else 0

        st.metric("긴급('상') 건수", f"{high_urgency_cnt}건")

    with col4:

        ai_avail_cnt = len(filtered_df[filtered_df['ai_handling'] == '전용AI가능']) if 'ai_handling' in filtered_df.columns else 0

        st.metric("AI 처리 가능", f"{ai_avail_cnt}건")

    st.markdown("---")

    # 4. 시각화 차트 섹션

    tab1, tab2 = st.tabs(["📈 분석 차트", "📋 데이터 테이블"])

    with tab1:

        # 첫 번째 행: 카테고리별 요청 건수 & 처리 상태 분포

        row1_col1, row1_col2 = st.columns(2)

        

        with row1_col1:

            if 'category' in filtered_df.columns:

                cat_df = filtered_df['category'].value_counts().reset_index()

                cat_df.columns = ['유형', '건수']

                fig_cat = px.bar(

                    cat_df, x='유형', y='건수', 

                    title="📌 요청 유형(Category)별 건수",

                    text='건수',

                    color='유형',

                    color_discrete_sequence=px.colors.qualitative.Pastel

                )

                fig_cat.update_traces(textposition='outside')

                st.plotly_chart(fig_cat, use_container_width=True)

        with row1_col2:

            if 'status' in filtered_df.columns:

                status_df = filtered_df['status'].value_counts().reset_index()

                status_df.columns = ['상태', '건수']

                fig_status = px.pie(

                    status_df, names='상태', values='건수', 

                    title="📌 처리 상태(Status) 비율",

                    hole=0.4,

                    color_discrete_sequence=px.colors.qualitative.Safe

                )

                st.plotly_chart(fig_status, use_container_width=True)

        # 두 번째 행: 긴급도 vs 상태 교차 분석 & AI 활용 분류

        row2_col1, row2_col2 = st.columns(2)

        with row2_col1:

            if 'urgency' in filtered_df.columns and 'status' in filtered_df.columns:

                urgency_status = filtered_df.groupby(['urgency', 'status']).size().reset_index(name='건수')

                fig_urg = px.bar(

                    urgency_status, x='urgency', y='건수', color='status',

                    title="📌 긴급도별 처리 현황",

                    barmode='group',

                    category_orders={'urgency': ['상', '보통', '하']}

                )

                st.plotly_chart(fig_urg, use_container_width=True)

        with row2_col2:

            if 'ai_handling' in filtered_df.columns:

                ai_df = filtered_df['ai_handling'].value_counts().reset_index()

                ai_df.columns = ['AI 활용 구분', '건수']

                fig_ai = px.bar(

                    ai_df, x='AI 활용 구분', y='건수',

                    title="📌 AI 활용 가능 여부(ai_handling)",

                    color='AI 활용 구분',

                    text='건수'

                )

                fig_ai.update_traces(textposition='outside')

                st.plotly_chart(fig_ai, use_container_width=True)

        # 세 번째 행: 일자별 요청 추이 (시계열)

        if 'request_date' in filtered_df.columns and len(filtered_df) > 0:

            timeline_df = filtered_df.groupby(filtered_df['request_date'].dt.date).size().reset_index(name='요청수')

            timeline_df.columns = ['날짜', '요청수']

            fig_time = px.line(

                timeline_df, x='날짜', y='요청수',

                title="📅 일자별 요청 건수 추이",

                markers=True

            )

            st.plotly_chart(fig_time, use_container_width=True)

    with tab2:

        st.subheader("📄 필터링된 원본 데이터")

        # 데이터프레임 스타일링 및 표시

        st.dataframe(filtered_df, use_container_width=True, hide_index=True)

        

        # 필터링 결과 다운로드 버튼

        csv_download = filtered_df.to_csv(index=False).encode('utf-8-sig')

        st.download_button(

            label="📥 필터링된 데이터 CSV 다운로드",

            data=csv_download,

            file_name="filtered_support_requests.csv",

            mime="text/csv"

        )

else:

    # 업로드 전 안내 화면

    st.info("👈 왼쪽 사이드바에서 CSV 파일을 업로드해주세요.")

    st.markdown("""

    ### 📌 지원 데이터 포맷 안내

    업로드할 CSV 파일은 다음과 같은 컬럼을 포함하고 있어야 대시보드가 정상 동작합니다.

    - `request_id`: 요청 고유 ID (예: REQ-001)

    - `request_date`: 요청 일자 (예: 2026-07-02)

    - `category`: 업무 유형 (예: 교육, IT, 문서, 인사 등)

    - `summary`: 요청 내용 요약

    - `urgency`: 긴급도 (상, 보통, 하)

    - `status`: 진행 상태 (대기, 처리중, 완료)

    - `ai_handling`: AI 지원 처리 분류 (전용AI가능, 비식별후, 사용금지 등)

    """)
