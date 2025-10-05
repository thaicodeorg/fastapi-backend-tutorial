# TMD 

```bash
pip install python-dotenv httpx
```

## สร้าง .env

```bash
# .env
TMD_FORECAST_TOKEN="YOUR_TMD_ACCESS_TOKEN" # แทนที่ด้วย ACCESS_TOKEN จริงของคุณ
```

## Project Structure

```bash
.
├── .env
├── main.py
└── app/
    ├── __init__.py
    ├── services/
    │   ├── __init__.py
    │   └── weather_service.py # โมดูลบริการหลัก
    └── api/
        ├── __init__.py
        └── weather_router.py # โมดูลสำหรับ Endpoints
```

## Code สำหรับ Services
```py title="app/services/weather_service.py"
# app/services/weather_service.py

import os
import httpx
import logging
from datetime import datetime, timedelta
from typing import Dict, Any, List
from dotenv import load_dotenv

# โหลดตัวแปร ENV
load_dotenv()
TMD_FORECAST_TOKEN = os.getenv("TMD_FORECAST_TOKEN")

logger = logging.getLogger(__name__)

# TMD API URLs
WEATHER_TODAY_URL = "https://data.tmd.go.th/api/WeatherToday/V2/?uid=api&ukey=api12345&format=json"
FORECAST_HOURLY_BASE_URL = "https://data.tmd.go.th/nwpapi/v1/forecast/location/hourly/at"

# Cache setup (ควรเปลี่ยนไปใช้ Redis หรือ memory cache ที่ดีกว่าใน Production)
cache: Dict[str, Any] = {}
CACHE_DURATION = 300  # 5 minutes

class WeatherService:
    """
    บริการสำหรับเรียกข้อมูลสภาพอากาศจาก TMD API และจัดการ Cache
    """
    
    # ------------------ Cache & Utility Methods ------------------
    #WEATHER_TODAY_URL = "https://data.tmd.go.th/api/WeatherToday/V2/?uid=api&ukey=api12345&format=json"
    #FORECAST_HOURLY_BASE_URL = "https://data.tmd.go.th/nwpapi/v1/forecast/location/hourly/at"

    # CACHE_DURATION = 300
    #load_dotenv()
    #TMD_FORECAST_TOKEN = os.getenv("TMD_FORECAST_TOKEN")
    
    def get_stations_list(self, data: Dict[str, Any]) -> List[Dict[str, Any]]:
        """Extract stations list from the data structure"""
        if not data:
            return []
        
        stations_data = data.get('Stations', {})
        if isinstance(stations_data, dict):
            station_item = stations_data.get('Station')
            if isinstance(station_item, list):
                return station_item
            elif isinstance(station_item, dict):
                return [station_item]
        elif isinstance(stations_data, list):
            return stations_data
        
        logger.warning("Could not find stations list in data structure")
        return []

    async def _fetch_weather_today_data(self) -> Dict[str, Any]:
        """Fetch today's weather data (Original API)"""
        async with httpx.AsyncClient(timeout=50) as client:
            try:
                logger.info(f"Fetching data from TMD API: {WEATHER_TODAY_URL}")
                headers = {
                    'User-Agent': 'FastAPI-TMD-Client/1.0',
                    'Accept': 'application/json',
                }
                response = await client.get(WEATHER_TODAY_URL, headers=headers)
                response.raise_for_status()
                return response.json()
            except httpx.HTTPStatusError as e:
                logger.error(f"TMD WeatherToday API Status Error: {e.response.status_code}")
                raise Exception(f"API Error: {e.response.status_code}")
            except httpx.RequestError as e:
                logger.error(f"TMD WeatherToday API Request Error: {e}")
                raise Exception(f"API Request Failed: {e}")
    
    # ------------------ Public Methods for WeatherToday API ------------------

    async def get_cached_weather_today(self) -> Dict[str, Any]:
        """Get data from cache if valid, otherwise fetch new data"""
        now = datetime.now()
        cache_key = "weather_today"
        
        cached = cache.get(cache_key)
        if cached and (now - cached['timestamp']).total_seconds() < CACHE_DURATION:
            logger.info("Returning cached data for weather_today")
            return cached['data']
        
        logger.info("Cache expired or empty, fetching new data for weather_today")
        try:
            data = await self._fetch_weather_today_data()
            cache[cache_key] = {'data': data, 'timestamp': now}
            logger.info("Data cached successfully for weather_today")
            return data
        except Exception as e:
            logger.error(f"Failed to fetch new data: {e}")
            if cached:
                logger.info("Returning expired cached data as fallback")
                return cached['data']
            raise Exception("Service Unavailable: Could not fetch initial data.") from e

    # ------------------ Public Methods for Forecast Hourly API ------------------
    
    async def get_hourly_forecast(self, lat: float, lon: float, fields: str, date: str, hour: int, duration: int) -> Dict[str, Any]:
        """
        Fetch hourly forecast data (New API)
        Requires TMD_FORECAST_TOKEN from .env
        logger.info(TMD_FORECAST_TOKEN)
        """
        if not TMD_FORECAST_TOKEN:
            raise ValueError("TMD_FORECAST_TOKEN is not set in environment.")
        
        params = {
            "lat": lat,
            "lon": lon,
            "fields": fields,
            "date": date,
            "hour": hour,
            "duration": duration,
        }

        headers = {
            'accept': 'application/json',
            'authorization':  f'Bearer {TMD_FORECAST_TOKEN}'
        }

        logger.info(headers)
        
        async with httpx.AsyncClient(timeout=30) as client:
            try:
                logger.info(f"Fetching hourly forecast for lat={lat}, lon={lon}")
                response = await client.get(
                    FORECAST_HOURLY_BASE_URL,
                    params=params,
                    headers=headers
                )
                response.raise_for_status()
                return response.json()
            except httpx.HTTPStatusError as e:
                logger.error(f"TMD Forecast API Status Error: {e.response.status_code}")
                # 401 Unauthorized likely due to bad token
                if e.response.status_code == 401:
                    raise Exception("Unauthorized: Check TMD_FORECAST_TOKEN in .env.")
                raise Exception(f"API Error: {e.response.status_code}")
            except httpx.RequestError as e:
                logger.error(f"TMD Forecast API Request Error: {e}")
                raise Exception(f"API Request Failed: {e}")


# สร้าง Instance ของ Service เพื่อนำไปใช้ใน Router
weather_service = WeatherService()
```

## Code Routers
- File นี้ำ จะทำหน้าสำหรับรับ Request 

```py title="app/api/weather_router.py"
# app/api/weather_router.py

from fastapi import APIRouter, HTTPException, Query, Depends
from app.services.weather_service import weather_service
from typing import Optional, List
from datetime import date as date_type

router = APIRouter(prefix="/v1", tags=["Weather"])

# Dependency Injection เพื่อเข้าถึง Service Instance
def get_service():
    return weather_service

# ------------------ Original WeatherToday Endpoints ------------------

@router.get("/weather")
async def get_all_weather_data(service: weather_service = Depends(get_service)):
    """Get all weather stations data (cached)"""
    try:
        data = await service.get_cached_weather_today()
        return data
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"Service Unavailable: {str(e)}")

@router.get("/weather/{station_id}")
async def get_weather_by_station(station_id: str, service: weather_service = Depends(get_service)):
    """Get specific station data by WMO station number"""
    try:
        data = await service.get_cached_weather_today()
        stations = service.get_stations_list(data)
        
        for station in stations:
            if str(station.get('WmoStationNumber', '')).lower() == station_id.lower():
                return station
        
        raise HTTPException(status_code=404, detail=f"Station {station_id} not found.")
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"Service Unavailable: {str(e)}")

# ------------------ New Hourly Forecast Endpoint ------------------

@router.get("/forecast/hourly")
async def get_hourly_forecast(
    lat: float = Query(..., description="Latitude, e.g., 13.10"),
    lon: float = Query(..., description="Longitude, e.g., 100.10"),
    fields: str = Query("tc,rh", description="Fields to retrieve (comma-separated), e.g., tc,rh"),
    date: date_type = Query(..., description="Date for forecast (YYYY-MM-DD), e.g., 2017-08-17"),
    hour: int = Query(8, ge=0, le=23, description="Start hour (0-23), e.g., 8"),
    duration: int = Query(2, ge=1, le=24, description="Duration in hours, e.g., 2"),
    service: weather_service = Depends(get_service)
):
    """
    Get hourly weather forecast data for a specific location.
    Requires 'authorization: Bearer TMD_FORECAST_TOKEN'
    """
    try:
        forecast_data = await service.get_hourly_forecast(
            lat=lat,
            lon=lon,
            fields=fields,
            date=date.isoformat(),
            hour=hour,
            duration=duration
        )
        return forecast_data
    except ValueError as e:
        raise HTTPException(status_code=500, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"Forecast Service Error: {str(e)}")

# สามารถเพิ่ม Endpoints อื่น ๆ จากโค้ดเดิมที่นี่
# เช่น /weather/province/{province}, /health, /cache/status เป็นต้น
```

```py title="main.py"
# main.py

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import logging
import uvicorn
from app.api.weather_router import router as weather_router
from app.services.weather_service import weather_service # เพื่อให้ cache ถูก initialize

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Thailand Weather API (Refactored)",
    description="API for fetching weather data from Thai Meteorological Department, structured with Services.",
    version="2.0.0"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# รวม Router
app.include_router(weather_router)

@app.get("/")
async def root():
    return {
        "message": "Thailand Weather API Refactored",
        "docs": "/docs",
        "new_endpoint_example": "/v1/forecast/hourly?lat=13.10&lon=100.10&date=2024-10-05"
    }

if __name__ == "__main__":
    # ใช้ weather_service.get_cached_weather_today() เพื่อโหลด cache ครั้งแรก
    # แต่เนื่องจากเป็น async function เราจะให้ uvicorn จัดการ (หรือใช้ on_event)
    logger.info("Starting up server. Cache will be loaded on first request.")
    uvicorn.run(app, host="0.0.0.0", port=8000)
```
## Globol variable และการอ้างอ้าง

"ประการนอก Class" (Defining variables outside of the class) can be tricky when those variables are meant to be used inside the class methods. This causes the common `AttributeError` you were seeing.

Based on the code structure you provided, the core issue is not that you *can't* define variables outside the class, but that you must know **how to access them** (Scope) and **why** they should be defined there (Purpose).

## 1\. ปัญหาหลัก: การเข้าถึง (The Scope Problem)

เมื่อคุณกำหนดตัวแปรภายนอก `class WeatherService:` ตัวแปรนั้นจะเป็น **Global Variable** (ตัวแปรส่วนกลาง)

เมื่อคุณพยายามเรียกใช้ตัวแปรนั้น **ภายใน method** ของ class (เช่น ใน `_fetch_weather_today_data` หรือ `get_cached_weather_today`) ด้วยคำว่า **`self.`** (เช่น `self.WEATHER_TODAY_URL` หรือ `self.CACHE_DURATION`) Python จะเข้าใจว่าคุณกำลังมองหาตัวแปรที่เป็น **Attribute** ของ Instance (Object) นั้น ๆ

เนื่องจากตัวแปรเหล่านั้นไม่ได้ถูกกำหนดไว้ใน `self.` แต่เป็น Global Variable จึงเกิดข้อผิดพลาด: **`'WeatherService' object has no attribute '...'`**

### วิธีแก้ไข (Fix):

ให้เลือกวิธีใดวิธีหนึ่งอย่างสม่ำเสมอ:

  * **Option A: ใช้เป็น Global Variable จริงๆ** (ถ้าตั้งใจให้เป็นตัวแปรส่วนกลางที่ทุกคนใช้ร่วมกัน):
    ให้เรียกใช้โดย **ไม่ต้องมี `self.`** นำหน้า เช่น:

    ```python
    # ใน Global Scope
    WEATHER_TODAY_URL = "..."

    # ใน method
    response = await client.get(WEATHER_TODAY_URL, headers=headers)
    ```

  * **Option B: กำหนดเป็น Class Attribute** (วิธีที่ดีที่สุดสำหรับค่าคงที่ของ Service):
    ให้กำหนดตัวแปรไว้ **ภายใน** Class โดยไม่ต้องอยู่ใน `__init__` และเรียกใช้ด้วย **`self.`** (ตามที่คุณได้แก้ไขไปแล้ว) เช่น:

    ```python
    class WeatherService:
        WEATHER_TODAY_URL = "..." # Attribute ของ Class

        async def _fetch_data(self):
            response = await client.get(self.WEATHER_TODAY_URL, headers=headers)
    ```

-----

## 2\. ตัวอย่างการใช้ตัวแปรนอก Class ที่ถูกต้อง (Global Variables)

ตัวแปรนอก Class ควรใช้สำหรับข้อมูลที่มีสถานะเป็น **Global State** หรือตัวแปรที่ทุกไฟล์หรือทุกส่วนของโปรแกรมต้องเข้าถึง โดยเฉพาะ **การตั้งค่า (Configuration)** และ **Cache**

ตัวอย่างที่คุณทำถูกต้องแล้วคือ:

### A. Global Setup (การตั้งค่าทั่วทั้งไฟล์)

  * **`TMD_FORECAST_TOKEN`**: ถูกต้องแล้วที่โหลดจาก `.env` นอก Class เพราะนี่คือค่าที่คุณต้องการเข้าถึงก่อนที่ Class จะถูกสร้าง หรือใช้ในบริบท Global ได้

### B. Global Shared State (สถานะที่ใช้ร่วมกัน)

  * **`cache: Dict[str, Any] = {}`**: ถูกต้องที่สุดที่ควรอยู่ **นอก Class** เพราะว่า `cache` นี้ควรเป็นพื้นที่เก็บข้อมูลร่วมกัน (Shared state) ที่ทุก Instance ของ `WeatherService` ใช้อ้างอิงร่วมกันทั้งหมด เพื่อให้แน่ใจว่าทุกคนอ่าน/เขียนจากที่เดียวกัน

ถ้าคุณย้าย `cache` เข้าไปอยู่ใน `__init__` (เป็น `self.cache`) ทุก Instance ใหม่ของ `WeatherService` จะมี Cache เป็นของตัวเอง ซึ่งจะทำให้ระบบ Cache ของคุณทำงานผิดพลาด ❌


## สร้าง instance
การสร้าง Instance นอก Class แบบนี้คือวิธีการมาตรฐานในการทำให้ Service Class ของคุณพร้อมใช้งานในแอปพลิเคชัน โดยเฉพาะอย่างยิ่งใน Framework ที่เป็น Asynchronous อย่าง **FastAPI**

## เหตุผลที่ต้องสร้าง Instance ⚙️

การเขียน `weather_service = WeatherService()` ที่ด้านล่างของไฟล์มีเหตุผลสำคัญดังนี้:

1.  **การเรียกใช้ใน Router (Dependency Injection):** ใน FastAPI, เมื่อคุณต้องการใช้ `WeatherService` ในฟังก์ชัน Router ของคุณ คุณจะใช้ Dependency Injection (`Depends`) โดยการอ้างถึง Instance ที่สร้างไว้แล้วนี้:

    ```python
    @router.get("/weather")
    async def get_weather(
        service: WeatherService = Depends(lambda: weather_service)
    ):
        # ... ใช้ service.get_cached_weather_today() ...
        pass
    ```

    การทำเช่นนี้ทำให้โค้ดของคุณเรียกใช้ **Instance เดียวกัน** ตลอดเวลา ซึ่งเป็นสิ่งสำคัญสำหรับ...

2.  **การจัดการทรัพยากร (Resource Management):**

      * **Cache (สำคัญมาก):** เนื่องจาก `cache: Dict[str, Any] = {}` ถูกกำหนดเป็น Global State (นอก Class แต่ถูกใช้ร่วมกัน) การสร้าง Instance เพียงครั้งเดียว (`weather_service`) จะทำให้ทุกฟังก์ชันในแอปพลิเคชันใช้ **Cache Dictionary ตัวเดียวกัน** ในหน่วยความจำ ถ้าคุณสร้าง Instance ใหม่ทุกครั้ง (เช่น `WeatherService()`) Cache จะไม่ทำงาน เพราะแต่ละ Instance จะมี Cache เปล่าของตัวเอง ❌
      * **Configuration:** Instance เดียวทำให้มั่นใจว่าค่า Config ต่าง ๆ เช่น URL ที่ถูกแก้ไขแล้ว (`self.FORECAST_HOURLY_BASE_URL`) และ Token (`self.TMD_FORECAST_TOKEN`) จะถูกโหลดและตั้งค่าเพียงครั้งเดียวเท่านั้น

3.  **ประสิทธิภาพ (Performance):** การหลีกเลี่ยงการสร้าง Object ใหม่ซ้ำๆ ทุกครั้งที่มี Request เข้ามาจะช่วยลด Overhead (ภาระงาน) และทำให้แอปพลิเคชันของคุณทำงานได้รวดเร็วขึ้น

**สรุป:** โค้ดที่ว่า **`weather_service = WeatherService()`** เป็นขั้นตอนที่ถูกต้องและจำเป็นในการทำให้ `WeatherService` ของคุณเป็น **Singleton Pattern** (มี Instance เดียว) ซึ่งเหมาะสมอย่างยิ่งสำหรับ Service Layer ในแอปพลิเคชัน Web API ครับ 👍