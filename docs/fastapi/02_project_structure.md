# Project Structure

## Basic  โครงสร้าง แยก โค้ดหลัก

```bash
pip install fastapi uvicorn httpx
```

แยกโค้ดหลักของ Application ไว้ใน package `app/`   
```bash
my_fastapi_project/
├── app/
│   ├── __init__.py      <-- เพื่อกำหนดให้ app เป็น Python Package
│   ├── main.py          <-- ไฟล์หลักที่สร้างและรวม FastAPI app
│   └── routers/
│       ├── __init__.py
│       └── user_router.py  <-- ตัวอย่างโมดูลสำหรับกำหนด Endpoint (/users)
│
└── run.py               <-- สคริปต์สำหรับรัน Uvicorn/FastAPI
```

```py title="app/main.py"
# app/main.py

from fastapi import FastAPI
from app.routers import user_router

def create_application() -> FastAPI:
    """
    สร้างและตั้งค่า FastAPI Application
    """
    # 1. สร้าง FastAPI Instance
    app = FastAPI(
        title="My Scalable FastAPI App",
        description="Example of a well-structured FastAPI project."
    )

    # 2. รวม Routers (Endpoints)
    # ใส่ทุก APIRouter ที่สร้างไว้ใน app/routers/
    app.include_router(user_router.router, prefix="/api/v1")

    # สามารถเพิ่มโค้ด Initialization อื่น ๆ ได้ที่นี่ เช่น event handlers, middleware

    return app

# สร้าง app instance เพื่อให้ uvicorn สามารถ import ไปใช้ได้
app = create_application()
```
- บรรทัดโค้ด from app.routers import user_router มีความหมายว่าเป็นการ นำเข้า (import) โมดูลที่ชื่อว่า user_router ซึ่งอยู่ที่ตำแหน่งภายในแพ็กเกจ app.routers 


### 1. ความหมายเชิงโครงสร้าง
- app: คือชื่อของ แพ็กเกจหลัก (main package) ของแอปพลิเคชันของคุณ

- .routers: คือชื่อของ แพ็กเกจย่อย (sub-package) ที่เก็บโค้ดส่วนที่ทำหน้าที่เป็นเราเตอร์ (APIRouter) ของ FastAPI

- user_router: คือชื่อของ โมดูล (มักจะเป็นไฟล์ user_router.py) ที่อยู่ภายในแพ็กเกจ app.routers/ ซึ่งไฟล์นี้จะเก็บตัวแปร router = APIRouter(...) เอาไว้

- ไฟล์นี้กำหนด Endpoint สำหรับ Feature

```py title="app/routers/user_router.py"
# app/routers/user_router.py

from fastapi import APIRouter

# สร้าง APIRouter
router = APIRouter(
    tags=["Users"] # ใช้สำหรับจัดกลุ่มใน Swagger UI
)

# Endpoint ตัวอย่าง
@router.get("/users/{user_id}")
async def read_user(user_id: int):
    return {"user_id": user_id, "name": "Test User"}

@router.post("/users/")
async def create_user(user: dict): # ควรใช้ Pydantic Model ในความเป็นจริง
    return {"message": "User created", "data": user}
```

```py title="run.py"
# run.py
import uvicorn

# กำหนดการตั้งค่า Server
# "app.main:app" หมายถึง:
# 1. "app.main" คือ module (ไฟล์ app/main.py)
# 2. ":app" คือตัวแปร FastAPI instance ที่ชื่อว่า 'app' ที่อยู่ในไฟล์นั้น
if __name__ == "__main__":
    uvicorn.run(
        "app.main:app",
        host="0.0.0.0",
        port=8000,
        reload=True  # ให้ reload อัตโนมัติเมื่อมีการแก้ไขโค้ด (สำหรับ Development)
    )
```

## Template 2 packages โมดูลย่อย เพื่อดึงข้อมูลจาก 2 External Api

- เพื่อดึงข้อมูลจาก 2 External APIs โดยใช้ FastAPI และไลบรารี httpx สำหรับการเรียก API แบบ Asynchronous (Async/Await) ซึ่งเป็นแนวทางปฏิบัติที่ดีที่สุดสำหรับ FastAPI

## Project Structure 

```bash
my_api_consumer/
├── app/
│   ├── __init__.py         # (1) กำหนดให้เป็น Python Package
│   ├── main.py             # (2) จุดรวม FastAPI App
│   ├── services/           # (3) แพ็กเกจสำหรับ Business Logic/การเรียก External API
│   │   ├── __init__.py
│   │   ├── service_a.py    # (4) โมดูลสำหรับเรียก API_A
│   │   └── service_b.py    # (5) โมดูลสำหรับเรียก API_B
│   └── api/                # (6) แพ็กเกจสำหรับ FastAPI Routers
│       ├── __init__.py
│       └── data_router.py  # (7) Endpoint ที่เรียกใช้ Service A & B
│
└── requirements.txt        # รายการ Dependencies (fastapi, uvicorn, httpx)
```

### 1. `app/services/service_a.py`

- สร้าง Services package และใช้ httpx.AsyncClient

```py title="app/services/service_a.py"

# app/services/service_a.py

import httpx
from typing import Dict, Any

API_A_BASE_URL = "https://api.example.com/products" # URL สมมติ

async def get_product_data(product_id: int) -> Dict[str, Any]:
    """ดึงข้อมูลสินค้าจาก API ภายนอก A"""
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(f"{API_A_BASE_URL}/{product_id}")
            response.raise_for_status() # ตรวจสอบ Error
            return response.json()
        except httpx.HTTPStatusError as e:
            return {"error": f"API A returned error: {e.response.status_code}"}
        except httpx.RequestError as e:
            return {"error": f"An error occurred while requesting API A: {e}"}
```

### 2. `app/services/service_b.py`

- สร้าง Services package และใช้ httpx.AsyncClient

```py title="app/services/service_b.py"

# app/services/service_b.py

import httpx
from typing import Dict, Any

API_B_BASE_URL = "https://api.another-example.com/pricing" # URL สมมติ

async def get_latest_price(item_code: str) -> Dict[str, Any]:
    """ดึงข้อมูลราคาจาก API ภายนอก B"""
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(f"{API_B_BASE_URL}/{item_code}")
            response.raise_for_status()
            return response.json()
        except httpx.HTTPStatusError as e:
            return {"error": f"API B returned error: {e.response.status_code}"}
        except httpx.RequestError as e:
            return {"error": f"An error occurred while requesting API B: {e}"}
```

### 3. The API Router Package
- Presentation layer  
```py title="app/api/data_router.py"
# app/api/data_router.py

from fastapi import APIRouter
from app.services.service_a import get_product_data
from app.services.service_b import get_latest_price
import asyncio
from typing import Dict, Any

router = APIRouter(tags=["Combined Data"])

@router.get("/combined-data/{id}")
async def get_combined_info(id: int):
    """
    Endpoint ที่เรียกใช้ Service A และ B พร้อมกัน (concurrently)
    เพื่อรวมข้อมูลสินค้าและราคา
    """
    item_code = f"ITEM-{id}" # สมมติว่า ID สินค้าคือ Item Code

    # ใช้ asyncio.gather เพื่อรันการเรียก API ทั้งสองพร้อมกัน
    # เพื่อลดเวลารอรวม
    product_task = get_product_data(id)
    price_task = get_latest_price(item_code)

    # รอให้ Task ทั้งสองเสร็จสมบูรณ์
    product_data, price_data = await asyncio.gather(product_task, price_task)

    return {
        "status": "success",
        "input_id": id,
        "product_info": product_data,
        "pricing_info": price_data,
        "note": "Data fetched concurrently from two external APIs."
    }
```

### C Main application
- Entry Point
```py title="app/main.py"
# app/main.py

from fastapi import FastAPI
from app.api.data_router import router as data_router # นำเข้าระบบ Router

app = FastAPI(
    title="API Consumer Example",
    description="Consumes two external APIs concurrently.",
    version="1.0.0"
)

# รวม Router เข้ากับ Main Application
app.include_router(data_router, prefix="/v1")

@app.get("/", include_in_schema=False)
async def root():
    return {"message": "Go to /docs for the API documentation."}
```

### Run Application
```
uvicorn app.main:app --reload
```