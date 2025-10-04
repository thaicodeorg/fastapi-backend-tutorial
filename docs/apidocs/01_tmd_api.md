# 01 TMD Api

## Target api

```
https://data.tmd.go.th/api/WeatherToday/V2/?uid=api&ukey=api12345&format=json
```
- Example Data จาก API  [tmd-data.json](./tdm-data.json)


1. ติดตั้ง Dependecies
```bash
pip install fastapi uvicorn requests python-multipart
```

2. code main.py
```python title="main.py"
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
import requests
from datetime import datetime, timedelta
import json
from typing import List, Optional
import logging

# Set up logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(
    title="Thailand Weather API",
    description="API for fetching weather data from Thai Meteorological Department",
    version="1.0.0"
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Cache setup
cache = {}
CACHE_DURATION = 300  # 5 minutes
TMD_API_URL = "https://data.tmd.go.th/api/WeatherToday/V2/?uid=api&ukey=api12345&format=json"

def fetch_weather_data():
    """Fetch data from TMD API with proper error handling"""
    try:
        logger.info(f"Fetching data from TMD API: {TMD_API_URL}")
        
        # Add headers to mimic a real browser
        headers = {
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36',
            'Accept': 'application/json',
            'Accept-Language': 'en-US,en;q=0.9',
        }
        
        response = requests.get(TMD_API_URL, headers=headers, timeout=50)
        logger.info(f"TMD API Response Status: {response.status_code}")
        
        if response.status_code != 200:
            logger.error(f"TMD API returned status code: {response.status_code}")
            logger.error(f"Response text: {response.text[:200]}...")
            raise Exception(f"TMD API returned status code: {response.status_code}")
        
        # Try to parse JSON
        try:
            data = response.json()
            logger.info("Successfully parsed JSON response")
            return data
        except json.JSONDecodeError as e:
            logger.error(f"JSON decode error: {e}")
            logger.error(f"Response content: {response.text[:500]}...")
            raise Exception(f"Invalid JSON response from TMD API: {e}")
            
    except requests.exceptions.Timeout:
        logger.error("Request timeout to TMD API")
        raise Exception("Request timeout to TMD API")
    except requests.exceptions.ConnectionError:
        logger.error("Connection error to TMD API")
        raise Exception("Cannot connect to TMD API - connection error")
    except requests.exceptions.RequestException as e:
        logger.error(f"Request exception: {e}")
        raise Exception(f"Request failed: {e}")
    except Exception as e:
        logger.error(f"Unexpected error: {e}")
        raise

def get_cached_data():
    """Get data from cache if valid, otherwise fetch new data"""
    now = datetime.now()
    
    if 'data' in cache and 'timestamp' in cache:
        time_diff = now - cache['timestamp']
        if time_diff.total_seconds() < CACHE_DURATION:
            logger.info("Returning cached data")
            return cache['data']
    
    # Fetch new data
    logger.info("Cache expired or empty, fetching new data")
    try:
        data = fetch_weather_data()
        cache['data'] = data
        cache['timestamp'] = now
        logger.info("Data cached successfully")
        return data
    except Exception as e:
        logger.error(f"Failed to fetch new data: {e}")
        # Return cached data even if expired if new fetch fails
        if 'data' in cache:
            logger.info("Returning expired cached data as fallback")
            return cache['data']
        raise e

def get_stations_list(data):
    """Extract stations list from the data structure"""
    if not data:
        return []
    
    # Try different possible structures
    stations_data = data.get('Stations', {})
    
    # Case 1: Stations is a dict with 'Station' key
    if isinstance(stations_data, dict):
        station_item = stations_data.get('Station')
        if isinstance(station_item, list):
            return station_item
        elif isinstance(station_item, dict):
            return [station_item]
    
    # Case 2: Stations is directly a list
    elif isinstance(stations_data, list):
        return stations_data
    
    # Case 3: Data might be directly the stations list
    elif isinstance(data, list):
        return data
    
    # Case 4: Try other possible keys
    for key in ['stations', 'data', 'Stations']:
        if key in data and isinstance(data[key], list):
            return data[key]
    
    logger.warning("Could not find stations list in data structure")
    return []

@app.get("/")
async def root():
    return {
        "message": "Thailand Weather API",
        "endpoints": {
            "/api/weather": "Get all weather stations data",
            "/api/weather/{station_id}": "Get specific station data",
            "/api/weather/province/{province}": "Get weather data by province",
            "/api/health": "Health check",
            "/api/cache/status": "Cache status",
            "/api/debug/raw": "Debug raw data (be careful - large response)"
        }
    }

@app.get("/api/weather")
async def get_all_weather_data():
    """Get all weather stations data"""
    try:
        data = get_cached_data()
        return data
    except Exception as e:
        logger.error(f"Error in get_all_weather_data: {e}")
        raise HTTPException(
            status_code=503, 
            detail=f"Unable to fetch weather data from TMD. Error: {str(e)}"
        )

@app.get("/api/weather/{station_id}")
async def get_weather_by_station(station_id: str):
    """Get specific station data by WMO station number"""
    try:
        data = get_cached_data()
        stations = get_stations_list(data)
        
        logger.info(f"Searching for station: {station_id}")
        logger.info(f"Total stations available: {len(stations)}")
        
        # Find station by WmoStationNumber
        for station in stations:
            if str(station.get('WmoStationNumber', '')) == str(station_id):
                return station
        
        # If not found, try case-insensitive search
        for station in stations:
            if str(station.get('WmoStationNumber', '')).lower() == str(station_id).lower():
                return station
        
        available_stations = [s.get('WmoStationNumber') for s in stations if s.get('WmoStationNumber')]
        raise HTTPException(
            status_code=404, 
            detail=f"Station {station_id} not found. Available stations: {available_stations[:10]}"  # Show first 10
        )
        
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error in get_weather_by_station: {e}")
        raise HTTPException(status_code=503, detail=f"Unable to fetch weather data: {str(e)}")

@app.get("/api/weather/province/{province}")
async def get_weather_by_province(province: str):
    """Get weather data by province"""
    try:
        data = get_cached_data()
        stations = get_stations_list(data)
        
        stations_in_province = []
        for station in stations:
            station_province = station.get('Province', '')
            if province.lower() in station_province.lower():
                stations_in_province.append(station)
        
        if not stations_in_province:
            available_provinces = list(set(s.get('Province') for s in stations if s.get('Province')))
            raise HTTPException(
                status_code=404, 
                detail=f"No stations found in province '{province}'. Available provinces: {available_provinces}"
            )
        
        return {
            "province": province,
            "count": len(stations_in_province),
            "stations": stations_in_province
        }
    except HTTPException:
        raise
    except Exception as e:
        logger.error(f"Error in get_weather_by_province: {e}")
        raise HTTPException(status_code=503, detail=f"Unable to fetch weather data: {str(e)}")

@app.get("/api/health")
async def health_check():
    """Health check endpoint"""
    try:
        # Test TMD API connection
        test_response = requests.get(TMD_API_URL, timeout=10)
        tmd_accessible = test_response.status_code == 200
        
        # Get cached data info
        cached_data = None
        stations_count = 0
        try:
            cached_data = get_cached_data()
            stations = get_stations_list(cached_data)
            stations_count = len(stations)
        except:
            pass
        
        return {
            "status": "healthy",
            "timestamp": datetime.now().isoformat(),
            "tmd_api_accessible": tmd_accessible,
            "tmd_api_status": test_response.status_code,
            "cached_data_available": cached_data is not None,
            "stations_in_cache": stations_count,
            "cache_age_seconds": (datetime.now() - cache.get('timestamp', datetime.now())).total_seconds() if cache.get('timestamp') else None
        }
    except Exception as e:
        return {
            "status": "unhealthy",
            "timestamp": datetime.now().isoformat(),
            "error": str(e),
            "tmd_api_accessible": False
        }

@app.get("/api/cache/status")
async def cache_status():
    """Check cache status"""
    if 'timestamp' in cache:
        time_diff = datetime.now() - cache['timestamp']
        data = cache.get('data', {})
        stations = get_stations_list(data)
        
        return {
            "cached": True,
            "last_updated": cache['timestamp'].isoformat(),
            "age_seconds": time_diff.total_seconds(),
            "cache_valid": time_diff.total_seconds() < CACHE_DURATION,
            "total_stations": len(stations),
            "sample_station_ids": [s.get('WmoStationNumber') for s in stations[:5]] if stations else []
        }
    return {
        "cached": False,
        "last_updated": None
    }

@app.get("/api/debug/raw")
async def debug_raw_data():
    """Debug endpoint to see raw data structure (use carefully)"""
    try:
        data = get_cached_data()
        return {
            "data_keys": list(data.keys()) if isinstance(data, dict) else type(data),
            "stations_structure": str(type(data.get('Stations'))) if isinstance(data, dict) else "N/A",
            "sample_data": str(data)[:1000] + "..." if len(str(data)) > 1000 else str(data)
        }
    except Exception as e:
        return {"error": str(e)}

@app.delete("/api/cache/clear")
async def clear_cache():
    """Clear the cache"""
    cache.clear()
    return {"message": "Cache cleared successfully"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)

```