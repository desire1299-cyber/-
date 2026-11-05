import os
import requests
import streamlit as st
from datetime import datetime, timedelta, timezone

# ---------- 설정 ----------
# data.go.kr에서 발급받은 "일반 인증키(Decoding)"를 넣으세요.
# 로컬: 환경변수 DATA_KEY / 배포: Streamlit secrets에 DATA_KEY 등록
API_KEY = os.getenv("DATA_KEY") or st.secrets.get("DATA_KEY", "")

# 지역: (기상청 격자 nx, ny, 에어코리아 측정소명)
LOCATIONS = {
    "서울 종로": (60, 127, "종로구"),
    "서울 강남": (61, 126, "강남구"),
    "부산 연제": (98, 76, "연산동"),
    "대구 중구": (89, 90, "남산동"),
    "대전 서구": (67, 100, "둔산동"),
}

KST = timezone(timedelta(hours=9))


# ---------- 기상청 단기예보 ----------
def get_base_time():
    now = datetime.now(KST) - timedelta(minutes=15)
    times = [2, 5, 8, 11, 14, 17, 20, 23]
    valid = [t for t in times if t <= now.hour]
    if valid:
        return now.strftime("%Y%m%d"), f"{valid[-1]:02d}00"
    yesterday = now - timedelta(days=1)
    return yesterday.strftime("%Y%m%d"), "2300"


@st.cache_data(ttl=600)
def get_weather(nx, ny):
    url = "http://apis.data.go.kr/1360000/VilageFcstInfoService_2.0/getVilageFcst"
    base_date, base_time = get_base_time()
    params = {
        "serviceKey": API_KEY, "pageNo": 1, "numOfRows": 300,
        "dataType": "JSON", "base_date": base_date, "base_time": base_time,
        "nx": nx, "ny": ny,
    }
    items = requests.get(url, params=params, timeout=10).json()[
        "response"]["body"]["items"]["item"]
    # 가장 가까운 예보 시각 값만 사용
    first_time = items[0]["fcstTime"]
    return {i["category"]: i["fcstValue"] for i in items if i["fcstTime"] == first_time}


# ---------- 에어코리아 ----------
@st.cache_data(ttl=600)
def get_air(station):
    url = "http://apis.data.go.kr/B552584/ArpltnInforInqireSvc/getMsrstnAcctoRltmMesureDnsty"
    params = {
        "serviceKey": API_KEY, "returnType": "json", "numOfRows": 1,
        "pageNo": 1, "stationName": station, "dataTerm": "DAILY", "ver": "1.0",
    }
    item = requests.get(url, params=params, timeout=10).json()[
        "response"]["body"]["items"][0]
    return item


# ---------- 추천 로직 (여기가 "AI" 부분, 나중에 모델로 교체 가능) ----------
def recommend(weather, air):
    pop = int(weather.get("POP", 0))   # 강수확률
    tmp = float(weather.get("TMP", 20))  # 기온
    pm25 = int(air.get("pm25Value") or 0)
    pm10 = int(air.get("pm10Value") or 0)

    umbrella = pop >= 40
    mask = pm25 >= 36 or pm10 >= 81
    if tmp <= 5:
        outer = "패딩 필수"
    elif tmp <= 15:
        outer = "겉옷 챙기기"
    elif tmp <= 22:
        outer = "가벼운 겉옷"
    else:
        outer = "반팔 OK"
    return umbrella, mask, outer, pop, tmp, pm25, pm10


# ---------- 화면 ----------
st.set_page_config(page_title="외출 준비 알림", page_icon="🚪")
st.title("🚪 외출 준비 알림")

if not API_KEY:
    st.error("DATA_KEY가 없어요. 환경변수나 secrets에 API 키를 넣어주세요.")
    st.stop()

place = st.selectbox("지역", list(LOCATIONS.keys()))
nx, ny, station = LOCATIONS[place]

try:
    weather = get_weather(nx, ny)
    air = get_air(station)
    umbrella, mask, outer, pop, tmp, pm25, pm10 = recommend(weather, air)

    c1, c2, c3 = st.columns(3)
    c1.metric("기온", f"{tmp:.0f}°C")
    c2.metric("강수확률", f"{pop}%")
    c3.metric("초미세먼지", f"{pm25}㎍/㎥")

    st.subheader("오늘 챙길 것")
    st.write("☔ 우산 챙기세요" if umbrella else "☀️ 우산 필요 없음")
    st.write("😷 마스크 쓰세요" if mask else "🙂 마스크 필요 없음")
    st.write(f"🧥 {outer}")
    st.caption(f"기준: {place} / 미세먼지 측정소: {station} / 출처: 기상청, 에어코리아")
except Exception as e:
    st.error(f"데이터를 불러오지 못했어요: {e}")
    st.info("키 활성화(발급 후 1~2시간), 활용신청 여부, 측정소명을 확인해보세요.")

