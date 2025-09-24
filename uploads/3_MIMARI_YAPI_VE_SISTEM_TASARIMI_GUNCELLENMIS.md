# 🏗️ INVESTMENT HUNTER V2.0 - MİMARİ YAPI VE SİSTEM TASARIMI
**Comprehensive System Architecture & Design Specifications - Updated with n8n v1.111.0**

## 🎯 **SİSTEM GENEL BAKIŞ**

### **Mimari Prensipler**
- **Microservices Architecture:** Modüler, scalable yapı
- **Event-Driven Design:** Real-time data flow
- **API-First Approach:** Decoupled frontend/backend
- **Database-Centric:** PostgreSQL as single source of truth
- **Automation-First:** n8n workflow automation (v1.111.0)

### **Teknoloji Stack'i**
```
Frontend: React 18 + TypeScript + Vite + Shadcn-ui + Tailwind CSS
Backend: Node.js + Express + PostgreSQL + Prisma ORM
Automation: n8n v1.111.0 (Self-hosted Docker + SQLite)
AI: OpenAI GPT-4 API
Real-time: WebSocket + Server-Sent Events
Caching: Redis (optional)
Deployment: Docker + VDS hosting
```

---

## 🗄️ **DATABASE TASARIMI**

### **PostgreSQL Schema**
```sql
-- Ana Vehicles Tablosu
CREATE TABLE vehicles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  listing_number VARCHAR UNIQUE NOT NULL,
  brand VARCHAR NOT NULL,
  model VARCHAR NOT NULL,
  year INTEGER NOT NULL,
  listing_price DECIMAL(12,2) NOT NULL,
  fair_price DECIMAL(12,2),
  quick_sale_price DECIMAL(12,2),
  quick_buy_price DECIMAL(12,2),
  mileage INTEGER,
  city VARCHAR NOT NULL,
  district VARCHAR,
  days_on_market INTEGER DEFAULT 0,
  price_drop_percent DECIMAL(5,2),
  price_drop_amount DECIMAL(12,2),
  description TEXT,
  image_url VARCHAR,
  damage_painted INTEGER DEFAULT 0,
  damage_replaced INTEGER DEFAULT 0,
  heavy_damage BOOLEAN DEFAULT FALSE,
  opportunity_score INTEGER CHECK (opportunity_score >= 0 AND opportunity_score <= 100),
  potential_profit DECIMAL(12,2),
  estimated_sale_days INTEGER,
  seller VARCHAR,
  seller_type VARCHAR CHECK (seller_type IN ('Galeri', 'Sahibinden', 'Ekspertiz')),
  is_todays_opportunity BOOLEAN DEFAULT FALSE,
  fuel_type VARCHAR,
  transmission VARCHAR,
  engine_size VARCHAR,
  horsepower VARCHAR,
  color VARCHAR,
  body_type VARCHAR,
  yearly_return DECIMAL(5,2),
  holding_period INTEGER,
  risk_level VARCHAR CHECK (risk_level IN ('Düşük', 'Orta', 'Yüksek')),
  tramer_amount DECIMAL(12,2),
  -- Normalization fields
  raw_data JSONB, -- Original scraped data
  normalized_at TIMESTAMP DEFAULT NOW(),
  data_source VARCHAR DEFAULT 'sahibinden',
  scrape_batch_id VARCHAR,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Raw Scraped Data Table (for normalization pipeline)
CREATE TABLE raw_vehicle_data (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  batch_id VARCHAR NOT NULL,
  source VARCHAR NOT NULL DEFAULT 'sahibinden',
  raw_json JSONB NOT NULL,
  processed BOOLEAN DEFAULT FALSE,
  normalized_vehicle_id UUID REFERENCES vehicles(id),
  error_message TEXT,
  scraped_at TIMESTAMP DEFAULT NOW(),
  processed_at TIMESTAMP
);

-- Package KM Data (for populate_package_km.py)
CREATE TABLE vehicle_packages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vehicle_id UUID REFERENCES vehicles(id) ON DELETE CASCADE,
  package_type VARCHAR NOT NULL, -- 'basic', 'comfort', 'premium'
  package_details JSONB,
  km_adjustment INTEGER DEFAULT 0,
  price_adjustment DECIMAL(12,2) DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Fiyat Geçmişi Tablosu
CREATE TABLE price_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vehicle_id UUID REFERENCES vehicles(id) ON DELETE CASCADE,
  price DECIMAL(12,2) NOT NULL,
  change_amount DECIMAL(12,2),
  change_percent DECIMAL(5,2),
  changed_at TIMESTAMP DEFAULT NOW(),
  source VARCHAR DEFAULT 'system',
  batch_id VARCHAR -- For tracking n8n batch updates
);

-- Kullanıcı Tablosu
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR UNIQUE NOT NULL,
  password_hash VARCHAR NOT NULL,
  role VARCHAR DEFAULT 'user' CHECK (role IN ('user', 'admin', 'analyst')),
  preferences JSONB DEFAULT '{}',
  created_at TIMESTAMP DEFAULT NOW(),
  last_login TIMESTAMP
);

-- Kullanıcı Favorileri
CREATE TABLE user_favorites (
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  vehicle_id UUID REFERENCES vehicles(id) ON DELETE CASCADE,
  created_at TIMESTAMP DEFAULT NOW(),
  notes TEXT,
  PRIMARY KEY (user_id, vehicle_id)
);

-- Fırsat Skorları Geçmişi
CREATE TABLE opportunity_scores_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vehicle_id UUID REFERENCES vehicles(id) ON DELETE CASCADE,
  score INTEGER NOT NULL,
  factors JSONB,
  calculated_at TIMESTAMP DEFAULT NOW()
);

-- n8n Workflow Execution Logs
CREATE TABLE n8n_execution_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workflow_id VARCHAR NOT NULL,
  execution_id VARCHAR NOT NULL,
  status VARCHAR NOT NULL, -- 'success', 'error', 'running'
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP,
  error_message TEXT,
  processed_records INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Sistem Logları
CREATE TABLE system_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  level VARCHAR NOT NULL CHECK (level IN ('info', 'warning', 'error', 'debug')),
  message TEXT NOT NULL,
  context JSONB,
  source VARCHAR DEFAULT 'system', -- 'n8n', 'api', 'normalization'
  created_at TIMESTAMP DEFAULT NOW()
);
```

### **Database Views**
```sql
-- Optimized Vehicle Opportunities View
CREATE VIEW vehicle_opportunities AS
SELECT 
  v.*,
  ph.latest_price,
  ph.price_trend,
  ph.last_updated,
  vp.package_count,
  CASE 
    WHEN v.opportunity_score >= 80 THEN 'Excellent'
    WHEN v.opportunity_score >= 60 THEN 'Good'
    WHEN v.opportunity_score >= 40 THEN 'Fair'
    ELSE 'Poor'
  END as opportunity_grade
FROM vehicles v
LEFT JOIN (
  SELECT DISTINCT ON (vehicle_id) 
    vehicle_id, 
    price as latest_price,
    change_percent as price_trend,
    changed_at as last_updated
  FROM price_history 
  ORDER BY vehicle_id, changed_at DESC
) ph ON v.id = ph.vehicle_id
LEFT JOIN (
  SELECT vehicle_id, COUNT(*) as package_count
  FROM vehicle_packages 
  GROUP BY vehicle_id
) vp ON v.id = vp.vehicle_id
WHERE v.opportunity_score > 30;

-- Normalization Pipeline Status View
CREATE VIEW normalization_status AS
SELECT 
  batch_id,
  source,
  COUNT(*) as total_records,
  COUNT(CASE WHEN processed THEN 1 END) as processed_records,
  COUNT(CASE WHEN error_message IS NOT NULL THEN 1 END) as error_records,
  MIN(scraped_at) as batch_start,
  MAX(processed_at) as batch_end
FROM raw_vehicle_data 
GROUP BY batch_id, source
ORDER BY batch_start DESC;
```

### **Database Indexes**
```sql
-- Performance Indexes
CREATE INDEX idx_vehicles_city ON vehicles(city);
CREATE INDEX idx_vehicles_brand ON vehicles(brand);
CREATE INDEX idx_vehicles_opportunity_score ON vehicles(opportunity_score DESC);
CREATE INDEX idx_vehicles_price ON vehicles(listing_price);
CREATE INDEX idx_vehicles_created_at ON vehicles(created_at DESC);
CREATE INDEX idx_vehicles_todays_opportunity ON vehicles(is_todays_opportunity) WHERE is_todays_opportunity = true;
CREATE INDEX idx_price_history_vehicle_id ON price_history(vehicle_id);
CREATE INDEX idx_price_history_changed_at ON price_history(changed_at DESC);

-- Normalization Pipeline Indexes
CREATE INDEX idx_raw_vehicle_data_batch_id ON raw_vehicle_data(batch_id);
CREATE INDEX idx_raw_vehicle_data_processed ON raw_vehicle_data(processed) WHERE processed = false;
CREATE INDEX idx_raw_vehicle_data_source ON raw_vehicle_data(source);

-- n8n Integration Indexes
CREATE INDEX idx_n8n_execution_logs_workflow_id ON n8n_execution_logs(workflow_id);
CREATE INDEX idx_n8n_execution_logs_status ON n8n_execution_logs(status);
CREATE INDEX idx_n8n_execution_logs_start_time ON n8n_execution_logs(start_time DESC);

-- Composite Indexes
CREATE INDEX idx_vehicles_city_brand ON vehicles(city, brand);
CREATE INDEX idx_vehicles_price_score ON vehicles(listing_price, opportunity_score);
CREATE INDEX idx_vehicles_search ON vehicles(city, brand, year, fuel_type);
```

---

## 🤖 **N8N AUTOMATION WORKFLOWS (v1.111.0)**

### **n8n Configuration**
```javascript
// n8n v1.111.0 Configuration
const N8N_CONFIG = {
  baseUrl: 'https://n8n.yatirimavcisi.com.tr/api/v1',
  version: '1.111.0',
  deployment: 'Self-hosted Docker',
  database: 'SQLite',
  proxy: 'Nginx with auto API key injection',
  healthEndpoint: '/health', // Updated in v1.111.0
  webhookEndpoint: '/webhook',
  workflowEndpoint: '/workflows'
};

// API Client Configuration
class N8nApiClient {
  constructor() {
    this.baseUrl = 'https://n8n.yatirimavcisi.com.tr/api/v1';
    this.headers = {
      'Content-Type': 'application/json',
      // API Key auto-injected by Nginx proxy
    };
  }

  async getHealth() {
    // Updated health check for v1.111.0
    const response = await fetch(`${this.baseUrl}/health`);
    return response.json();
  }

  async executeWorkflow(workflowId, data) {
    const response = await fetch(`${this.baseUrl}/workflows/${workflowId}/execute`, {
      method: 'POST',
      headers: this.headers,
      body: JSON.stringify(data)
    });
    return response.json();
  }

  async getWorkflowExecutions(workflowId) {
    const response = await fetch(`${this.baseUrl}/executions?workflowId=${workflowId}`);
    return response.json();
  }
}
```

### **Workflow 1: Vehicle Data Collection & Normalization**
```json
{
  "name": "Vehicle Data Collector v2.0",
  "active": true,
  "version": "1.111.0",
  "nodes": [
    {
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.cron",
      "parameters": {
        "rule": {
          "interval": [{"field": "hours", "value": 6}]
        }
      }
    },
    {
      "name": "Fetch Sahibinden Data",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "url": "https://api.sahibinden.com/vehicles",
        "method": "GET",
        "headers": {
          "User-Agent": "Investment Hunter Bot v2.0"
        }
      }
    },
    {
      "name": "Store Raw Data",
      "type": "n8n-nodes-base.postgres",
      "parameters": {
        "operation": "insert",
        "table": "raw_vehicle_data",
        "columns": "batch_id,source,raw_json,scraped_at"
      }
    },
    {
      "name": "Trigger Normalization",
      "type": "n8n-nodes-base.function",
      "parameters": {
        "functionCode": `
          // Trigger Python normalization script
          const batchId = $input.first().json.batch_id;
          
          // Call normalize-scrapeit-corrected.py
          const normalizeCommand = \`python3 /scripts/normalize-scrapeit-corrected.py --batch-id=\${batchId}\`;
          
          // Call populate_package_km.py
          const packageCommand = \`python3 /scripts/populate_package_km.py --batch-id=\${batchId}\`;
          
          return [{
            batch_id: batchId,
            normalize_command: normalizeCommand,
            package_command: packageCommand,
            status: 'triggered'
          }];
        `
      }
    },
    {
      "name": "Execute Normalization Scripts",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "={{$json.normalize_command}}"
      }
    },
    {
      "name": "Execute Package KM Scripts",
      "type": "n8n-nodes-base.executeCommand",
      "parameters": {
        "command": "={{$json.package_command}}"
      }
    },
    {
      "name": "Calculate Opportunity Scores",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "url": "http://localhost:3001/api/internal/calculate-scores",
        "method": "POST",
        "body": {
          "batch_id": "={{$json.batch_id}}"
        }
      }
    },
    {
      "name": "Log Execution",
      "type": "n8n-nodes-base.postgres",
      "parameters": {
        "operation": "insert",
        "table": "n8n_execution_logs",
        "columns": "workflow_id,execution_id,status,start_time,end_time,processed_records"
      }
    },
    {
      "name": "Trigger Frontend Webhook",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "httpMethod": "POST",
        "path": "vehicle-updates",
        "responseMode": "onReceived"
      }
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [["Fetch Sahibinden Data"]]
    },
    "Fetch Sahibinden Data": {
      "main": [["Store Raw Data"]]
    },
    "Store Raw Data": {
      "main": [["Trigger Normalization"]]
    },
    "Trigger Normalization": {
      "main": [["Execute Normalization Scripts"]]
    },
    "Execute Normalization Scripts": {
      "main": [["Execute Package KM Scripts"]]
    },
    "Execute Package KM Scripts": {
      "main": [["Calculate Opportunity Scores"]]
    },
    "Calculate Opportunity Scores": {
      "main": [["Log Execution", "Trigger Frontend Webhook"]]
    }
  }
}
```

### **Workflow 2: Real-time Price Monitor**
```json
{
  "name": "Price Monitor v2.0",
  "active": true,
  "version": "1.111.0",
  "nodes": [
    {
      "name": "Database Query",
      "type": "n8n-nodes-base.postgres",
      "parameters": {
        "operation": "select",
        "query": "SELECT * FROM vehicles WHERE updated_at < NOW() - INTERVAL '2 hours' LIMIT 100"
      }
    },
    {
      "name": "Check Current Prices",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "url": "https://api.sahibinden.com/vehicle/{{$json.listing_number}}",
        "method": "GET"
      }
    },
    {
      "name": "Compare Prices",
      "type": "n8n-nodes-base.function",
      "parameters": {
        "functionCode": `
          const oldPrice = $input.first().json.listing_price;
          const newPrice = $input.last().json.price;
          const changePercent = ((newPrice - oldPrice) / oldPrice) * 100;
          
          return [{
            vehicleId: $input.first().json.id,
            listingNumber: $input.first().json.listing_number,
            oldPrice,
            newPrice,
            changePercent,
            changeAmount: newPrice - oldPrice,
            hasChanged: Math.abs(changePercent) > 1,
            timestamp: new Date().toISOString()
          }];
        `
      }
    },
    {
      "name": "Update Vehicle Price",
      "type": "n8n-nodes-base.postgres",
      "parameters": {
        "operation": "update",
        "table": "vehicles",
        "updateKey": "id",
        "columns": "listing_price,updated_at"
      }
    },
    {
      "name": "Insert Price History",
      "type": "n8n-nodes-base.postgres",
      "parameters": {
        "operation": "insert",
        "table": "price_history",
        "columns": "vehicle_id,price,change_amount,change_percent,source,batch_id"
      }
    },
    {
      "name": "Send Real-time Notification",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "httpMethod": "POST",
        "path": "price-changes",
        "responseMode": "onReceived"
      }
    }
  ]
}
```

### **Workflow 3: Health Check & Monitoring**
```json
{
  "name": "System Health Monitor",
  "active": true,
  "version": "1.111.0",
  "nodes": [
    {
      "name": "Ping Trigger",
      "type": "n8n-nodes-base.cron",
      "parameters": {
        "rule": {
          "interval": [{"field": "minutes", "value": 15}]
        }
      }
    },
    {
      "name": "Check Database",
      "type": "n8n-nodes-base.postgres",
      "parameters": {
        "operation": "select",
        "query": "SELECT COUNT(*) as vehicle_count FROM vehicles"
      }
    },
    {
      "name": "Check API Endpoints",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "url": "http://localhost:3001/api/health",
        "method": "GET"
      }
    },
    {
      "name": "Log System Status",
      "type": "n8n-nodes-base.postgres",
      "parameters": {
        "operation": "insert",
        "table": "system_logs",
        "columns": "level,message,context,source"
      }
    }
  ]
}
```

---

## 📄 **DATA NORMALIZATION SCRIPTS**

### **normalize-scrapeit-corrected.py Integration**
```python
#!/usr/bin/env python3
"""
Enhanced normalize-scrapeit-corrected.py for Investment Hunter v2.0
Integrates with PostgreSQL and n8n workflows
"""

import json
import sys
import argparse
import psycopg2
from datetime import datetime
import logging
from typing import Dict, List, Optional

class VehicleDataNormalizer:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        self.conn = None
        self.setup_logging()
    
    def setup_logging(self):
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s'
        )
        self.logger = logging.getLogger(__name__)
    
    def connect_db(self):
        """Connect to PostgreSQL database"""
        try:
            self.conn = psycopg2.connect(**self.db_config)
            self.logger.info("Database connection established")
        except Exception as e:
            self.logger.error(f"Database connection failed: {e}")
            raise
    
    def get_raw_data(self, batch_id: str) -> List[Dict]:
        """Fetch raw vehicle data for processing"""
        cursor = self.conn.cursor()
        cursor.execute(
            "SELECT id, raw_json FROM raw_vehicle_data WHERE batch_id = %s AND processed = FALSE",
            (batch_id,)
        )
        return cursor.fetchall()
    
    def normalize_vehicle_data(self, raw_data: Dict) -> Dict:
        """
        Normalize raw scraped vehicle data
        Enhanced version of original normalize-scrapeit-corrected.py logic
        """
        normalized = {
            'listing_number': self.extract_listing_number(raw_data),
            'brand': self.normalize_brand(raw_data.get('brand', '')),
            'model': self.normalize_model(raw_data.get('model', '')),
            'year': self.extract_year(raw_data),
            'listing_price': self.extract_price(raw_data),
            'mileage': self.extract_mileage(raw_data),
            'city': self.normalize_city(raw_data.get('city', '')),
            'district': raw_data.get('district', ''),
            'fuel_type': self.normalize_fuel_type(raw_data.get('fuel_type', '')),
            'transmission': self.normalize_transmission(raw_data.get('transmission', '')),
            'color': raw_data.get('color', ''),
            'seller_type': self.determine_seller_type(raw_data),
            'description': raw_data.get('description', ''),
            'image_url': raw_data.get('image_url', ''),
            'damage_painted': self.extract_damage_painted(raw_data),
            'damage_replaced': self.extract_damage_replaced(raw_data),
            'heavy_damage': self.detect_heavy_damage(raw_data),
            'raw_data': raw_data,
            'data_source': 'sahibinden',
            'normalized_at': datetime.now().isoformat()
        }
        
        return normalized
    
    def extract_listing_number(self, data: Dict) -> str:
        """Extract and validate listing number"""
        listing_num = data.get('listing_number', data.get('id', ''))
        return str(listing_num).strip()
    
    def normalize_brand(self, brand: str) -> str:
        """Normalize vehicle brand names"""
        brand_mapping = {
            'VOLKSWAGEN': 'Volkswagen',
            'MERCEDES-BENZ': 'Mercedes-Benz',
            'BMW': 'BMW',
            'AUDI': 'Audi',
            'TOYOTA': 'Toyota',
            'HONDA': 'Honda',
            'FORD': 'Ford',
            'OPEL': 'Opel',
            'RENAULT': 'Renault',
            'PEUGEOT': 'Peugeot'
        }
        return brand_mapping.get(brand.upper(), brand.title())
    
    def extract_price(self, data: Dict) -> float:
        """Extract and clean price data"""
        price_str = str(data.get('price', '0'))
        # Remove currency symbols and spaces
        price_clean = ''.join(filter(str.isdigit, price_str))
        return float(price_clean) if price_clean else 0.0
    
    def extract_mileage(self, data: Dict) -> int:
        """Extract and clean mileage data"""
        mileage_str = str(data.get('mileage', '0'))
        mileage_clean = ''.join(filter(str.isdigit, mileage_str))
        return int(mileage_clean) if mileage_clean else 0
    
    def insert_normalized_data(self, normalized_data: Dict) -> str:
        """Insert normalized data into vehicles table"""
        cursor = self.conn.cursor()
        
        insert_query = """
        INSERT INTO vehicles (
            listing_number, brand, model, year, listing_price, mileage,
            city, district, fuel_type, transmission, color, seller_type,
            description, image_url, damage_painted, damage_replaced, 
            heavy_damage, raw_data, data_source, normalized_at
        ) VALUES (
            %(listing_number)s, %(brand)s, %(model)s, %(year)s, %(listing_price)s,
            %(mileage)s, %(city)s, %(district)s, %(fuel_type)s, %(transmission)s,
            %(color)s, %(seller_type)s, %(description)s, %(image_url)s,
            %(damage_painted)s, %(damage_replaced)s, %(heavy_damage)s,
            %(raw_data)s, %(data_source)s, %(normalized_at)s
        ) RETURNING id
        """
        
        cursor.execute(insert_query, normalized_data)
        vehicle_id = cursor.fetchone()[0]
        self.conn.commit()
        
        return vehicle_id
    
    def mark_as_processed(self, raw_id: str, vehicle_id: str):
        """Mark raw data as processed"""
        cursor = self.conn.cursor()
        cursor.execute(
            "UPDATE raw_vehicle_data SET processed = TRUE, normalized_vehicle_id = %s, processed_at = NOW() WHERE id = %s",
            (vehicle_id, raw_id)
        )
        self.conn.commit()
    
    def process_batch(self, batch_id: str):
        """Process entire batch of raw data"""
        self.logger.info(f"Processing batch: {batch_id}")
        
        raw_records = self.get_raw_data(batch_id)
        processed_count = 0
        error_count = 0
        
        for raw_id, raw_json in raw_records:
            try:
                normalized_data = self.normalize_vehicle_data(raw_json)
                vehicle_id = self.insert_normalized_data(normalized_data)
                self.mark_as_processed(raw_id, vehicle_id)
                processed_count += 1
                
            except Exception as e:
                self.logger.error(f"Error processing record {raw_id}: {e}")
                error_count += 1
                
                # Mark as error
                cursor = self.conn.cursor()
                cursor.execute(
                    "UPDATE raw_vehicle_data SET error_message = %s WHERE id = %s",
                    (str(e), raw_id)
                )
                self.conn.commit()
        
        self.logger.info(f"Batch processing complete. Processed: {processed_count}, Errors: {error_count}")
        return processed_count, error_count

def main():
    parser = argparse.ArgumentParser(description='Normalize vehicle data')
    parser.add_argument('--batch-id', required=True, help='Batch ID to process')
    parser.add_argument('--db-host', default='localhost', help='Database host')
    parser.add_argument('--db-port', default='5432', help='Database port')
    parser.add_argument('--db-name', default='investment_hunter', help='Database name')
    parser.add_argument('--db-user', default='postgres', help='Database user')
    parser.add_argument('--db-password', required=True, help='Database password')
    
    args = parser.parse_args()
    
    db_config = {
        'host': args.db_host,
        'port': args.db_port,
        'database': args.db_name,
        'user': args.db_user,
        'password': args.db_password
    }
    
    normalizer = VehicleDataNormalizer(db_config)
    normalizer.connect_db()
    
    try:
        processed, errors = normalizer.process_batch(args.batch_id)
        print(f"SUCCESS: Processed {processed} records, {errors} errors")
        sys.exit(0)
    except Exception as e:
        print(f"FAILED: {e}")
        sys.exit(1)
    finally:
        if normalizer.conn:
            normalizer.conn.close()

if __name__ == "__main__":
    main()
```

### **populate_package_km.py Integration**
```python
#!/usr/bin/env python3
"""
Enhanced populate_package_km.py for Investment Hunter v2.0
Processes vehicle package and KM data
"""

import json
import sys
import argparse
import psycopg2
from datetime import datetime
import logging
from typing import Dict, List, Optional

class VehiclePackageProcessor:
    def __init__(self, db_config: Dict):
        self.db_config = db_config
        self.conn = None
        self.setup_logging()
    
    def setup_logging(self):
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s'
        )
        self.logger = logging.getLogger(__name__)
    
    def connect_db(self):
        """Connect to PostgreSQL database"""
        try:
            self.conn = psycopg2.connect(**self.db_config)
            self.logger.info("Database connection established")
        except Exception as e:
            self.logger.error(f"Database connection failed: {e}")
            raise
    
    def get_vehicles_for_batch(self, batch_id: str) -> List[Dict]:
        """Get vehicles that need package processing"""
        cursor = self.conn.cursor()
        cursor.execute("""
            SELECT v.id, v.listing_number, v.brand, v.model, v.year, 
                   v.mileage, v.raw_data
            FROM vehicles v
            JOIN raw_vehicle_data r ON r.normalized_vehicle_id = v.id
            WHERE r.batch_id = %s AND r.processed = TRUE
        """, (batch_id,))
        
        columns = [desc[0] for desc in cursor.description]
        return [dict(zip(columns, row)) for row in cursor.fetchall()]
    
    def extract_package_info(self, raw_data: Dict) -> List[Dict]:
        """Extract package information from raw data"""
        packages = []
        
        # Extract standard packages
        if 'packages' in raw_data:
            for package_data in raw_data['packages']:
                packages.append({
                    'package_type': self.normalize_package_type(package_data.get('type', '')),
                    'package_details': package_data,
                    'price_adjustment': self.calculate_price_adjustment(package_data),
                    'km_adjustment': self.calculate_km_adjustment(package_data)
                })
        
        # Extract from description if packages not explicitly listed
        description = raw_data.get('description', '').lower()
        if 'paket' in description or 'donanım' in description:
            inferred_packages = self.infer_packages_from_description(description)
            packages.extend(inferred_packages)
        
        return packages
    
    def normalize_package_type(self, package_type: str) -> str:
        """Normalize package type names"""
        package_mapping = {
            'temel': 'basic',
            'basic': 'basic',
            'konfor': 'comfort',
            'comfort': 'comfort',
            'premium': 'premium',
            'lux': 'premium',
            'luxury': 'premium',
            'sport': 'sport',
            'spor': 'sport'
        }
        return package_mapping.get(package_type.lower(), 'unknown')
    
    def calculate_price_adjustment(self, package_data: Dict) -> float:
        """Calculate price adjustment based on package"""
        base_adjustments = {
            'basic': 0,
            'comfort': 5000,
            'premium': 15000,
            'sport': 20000
        }
        
        package_type = self.normalize_package_type(package_data.get('type', ''))
        base_adjustment = base_adjustments.get(package_type, 0)
        
        # Additional adjustments based on specific features
        features = package_data.get('features', [])
        feature_adjustments = {
            'sunroof': 3000,
            'leather': 4000,
            'navigation': 2000,
            'premium_sound': 2500,
            'xenon': 1500
        }
        
        total_adjustment = base_adjustment
        for feature in features:
            if feature.lower() in feature_adjustments:
                total_adjustment += feature_adjustments[feature.lower()]
        
        return total_adjustment
    
    def calculate_km_adjustment(self, package_data: Dict) -> int:
        """Calculate KM adjustment based on package wear"""
        # Premium packages might indicate better maintenance
        package_type = self.normalize_package_type(package_data.get('type', ''))
        
        km_adjustments = {
            'basic': 0,
            'comfort': -5000,  # Better maintained, effective lower KM
            'premium': -10000,
            'sport': 5000      # Sport packages might be driven harder
        }
        
        return km_adjustments.get(package_type, 0)
    
    def infer_packages_from_description(self, description: str) -> List[Dict]:
        """Infer package information from vehicle description"""
        packages = []
        
        # Keywords that indicate different package levels
        if any(word in description for word in ['premium', 'lux', 'luxury', 'full']):
            packages.append({
                'package_type': 'premium',
                'package_details': {'inferred': True, 'source': 'description'},
                'price_adjustment': 15000,
                'km_adjustment': -10000
            })
        elif any(word in description for word in ['konfor', 'comfort', 'orta']):
            packages.append({
                'package_type': 'comfort',
                'package_details': {'inferred': True, 'source': 'description'},
                'price_adjustment': 5000,
                'km_adjustment': -5000
            })
        
        return packages
    
    def insert_vehicle_packages(self, vehicle_id: str, packages: List[Dict]):
        """Insert vehicle package data"""
        cursor = self.conn.cursor()
        
        for package in packages:
            insert_query = """
            INSERT INTO vehicle_packages (
                vehicle_id, package_type, package_details, 
                km_adjustment, price_adjustment
            ) VALUES (%s, %s, %s, %s, %s)
            """
            
            cursor.execute(insert_query, (
                vehicle_id,
                package['package_type'],
                json.dumps(package['package_details']),
                package['km_adjustment'],
                package['price_adjustment']
            ))
        
        self.conn.commit()
    
    def update_vehicle_with_adjustments(self, vehicle_id: str, packages: List[Dict]):
        """Update vehicle with package-based adjustments"""
        total_price_adjustment = sum(p['price_adjustment'] for p in packages)
        total_km_adjustment = sum(p['km_adjustment'] for p in packages)
        
        cursor = self.conn.cursor()
        
        # Update vehicle with adjusted values
        cursor.execute("""
            UPDATE vehicles 
            SET 
                mileage = GREATEST(0, mileage + %s),
                listing_price = listing_price + %s,
                updated_at = NOW()
            WHERE id = %s
        """, (total_km_adjustment, total_price_adjustment, vehicle_id))
        
        self.conn.commit()
    
    def process_batch(self, batch_id: str):
        """Process package data for entire batch"""
        self.logger.info(f"Processing package data for batch: {batch_id}")
        
        vehicles = self.get_vehicles_for_batch(batch_id)
        processed_count = 0
        error_count = 0
        
        for vehicle in vehicles:
            try:
                packages = self.extract_package_info(vehicle['raw_data'])
                
                if packages:
                    self.insert_vehicle_packages(vehicle['id'], packages)
                    self.update_vehicle_with_adjustments(vehicle['id'], packages)
                    self.logger.info(f"Processed {len(packages)} packages for vehicle {vehicle['listing_number']}")
                
                processed_count += 1
                
            except Exception as e:
                self.logger.error(f"Error processing vehicle {vehicle['id']}: {e}")
                error_count += 1
        
        self.logger.info(f"Package processing complete. Processed: {processed_count}, Errors: {error_count}")
        return processed_count, error_count

def main():
    parser = argparse.ArgumentParser(description='Process vehicle package data')
    parser.add_argument('--batch-id', required=True, help='Batch ID to process')
    parser.add_argument('--db-host', default='localhost', help='Database host')
    parser.add_argument('--db-port', default='5432', help='Database port')
    parser.add_argument('--db-name', default='investment_hunter', help='Database name')
    parser.add_argument('--db-user', default='postgres', help='Database user')
    parser.add_argument('--db-password', required=True, help='Database password')
    
    args = parser.parse_args()
    
    db_config = {
        'host': args.db_host,
        'port': args.db_port,
        'database': args.db_name,
        'user': args.db_user,
        'password': args.db_password
    }
    
    processor = VehiclePackageProcessor(db_config)
    processor.connect_db()
    
    try:
        processed, errors = processor.process_batch(args.batch_id)
        print(f"SUCCESS: Processed {processed} vehicles, {errors} errors")
        sys.exit(0)
    except Exception as e:
        print(f"FAILED: {e}")
        sys.exit(1)
    finally:
        if processor.conn:
            processor.conn.close()

if __name__ == "__main__":
    main()
```

---

## ⚙️ **ENVIRONMENT CONFIGURATION**

### **.env Configuration**
```bash
# Application
NODE_ENV=development
PORT=3000
APP_URL=http://localhost:3000

# Database
DATABASE_URL=postgresql://username:password@localhost:5432/investment_hunter
DB_HOST=localhost
DB_PORT=5432
DB_NAME=investment_hunter
DB_USER=postgres
DB_PASSWORD=your_password
DB_SSL=false

# OpenAI API
VITE_OPENAI_API_KEY=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
OPENAI_MODEL=gpt-4
OPENAI_MAX_TOKENS=2000
OPENAI_TEMPERATURE=0.7

# n8n Integration (v1.111.0)
N8N_BASE_URL=https://n8n.yatirimavcisi.com.tr/api/v1
N8N_VERSION=1.111.0
N8N_DEPLOYMENT=docker-sqlite
N8N_WEBHOOK_URL=https://n8n.yatirimavcisi.com.tr/webhook/
N8N_API_KEY_AUTO_INJECT=true
N8N_HEALTH_ENDPOINT=/health
N8N_WORKFLOW_ENDPOINT=/workflows

# API Configuration
API_BASE_URL=http://localhost:3001/api
PROXY_API_URL=http://your-vds-ip:3001/proxy
API_TIMEOUT=10000
API_RETRY_ATTEMPTS=3

# Data Processing Scripts
NORMALIZE_SCRIPT_PATH=/scripts/normalize-scrapeit-corrected.py
PACKAGE_SCRIPT_PATH=/scripts/populate_package_km.py
PYTHON_EXECUTABLE=python3

# Redis (Optional)
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=your-redis-password

# JWT
JWT_SECRET=your-super-secret-jwt-key
JWT_EXPIRES_IN=7d

# File Upload
UPLOAD_MAX_SIZE=10485760
UPLOAD_ALLOWED_TYPES=jpg,jpeg,png,webp

# Monitoring
LOG_LEVEL=info
SENTRY_DSN=your-sentry-dsn
ANALYTICS_ID=your-analytics-id

# Feature Flags
ENABLE_REAL_TIME=true
ENABLE_CACHING=true
ENABLE_ANALYTICS=true
ENABLE_NOTIFICATIONS=true
ENABLE_N8N_INTEGRATION=true
ENABLE_DATA_NORMALIZATION=true

# External APIs
SAHIBINDEN_API_URL=https://api.sahibinden.com
SAHIBINDEN_API_KEY=your-sahibinden-api-key
RATE_LIMIT_REQUESTS=100
RATE_LIMIT_WINDOW=900000

# Security
CORS_ORIGIN=http://localhost:3000
HELMET_ENABLED=true
RATE_LIMITING_ENABLED=true

# Batch Processing
BATCH_SIZE=100
MAX_CONCURRENT_BATCHES=3
BATCH_TIMEOUT=300000
```

### **n8n Integration Service**
```typescript
// src/services/n8n/N8nService.ts
import { N8nApiClient } from './N8nApiClient';

export class N8nService {
  private client: N8nApiClient;
  
  constructor() {
    this.client = new N8nApiClient({
      baseUrl: process.env.N8N_BASE_URL!,
      version: process.env.N8N_VERSION!,
      autoInjectApiKey: process.env.N8N_API_KEY_AUTO_INJECT === 'true'
    });
  }
  
  async checkHealth(): Promise<boolean> {
    try {
      const health = await this.client.getHealth();
      return health.status === 'ok';
    } catch (error) {
      console.error('n8n health check failed:', error);
      return false;
    }
  }
  
  async triggerDataCollection(): Promise<string> {
    const workflowId = 'vehicle-data-collector-v2';
    const execution = await this.client.executeWorkflow(workflowId, {
      trigger: 'manual',
      timestamp: new Date().toISOString()
    });
    
    return execution.id;
  }
  
  async getExecutionStatus(executionId: string): Promise<string> {
    const execution = await this.client.getExecution(executionId);
    return execution.status;
  }
  
  async subscribeToWebhook(endpoint: string, callback: Function): Promise<void> {
    // WebSocket connection for real-time updates
    const ws = new WebSocket(`${this.client.baseUrl.replace('http', 'ws')}/webhook/${endpoint}`);
    
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      callback(data);
    };
    
    ws.onerror = (error) => {
      console.error('n8n webhook error:', error);
    };
  }
}

// Usage in React components
export const useN8nIntegration = () => {
  const [isHealthy, setIsHealthy] = useState(false);
  const [executions, setExecutions] = useState<any[]>([]);
  
  const n8nService = new N8nService();
  
  useEffect(() => {
    const checkHealth = async () => {
      const healthy = await n8nService.checkHealth();
      setIsHealthy(healthy);
    };
    
    checkHealth();
    const interval = setInterval(checkHealth, 60000); // Check every minute
    
    return () => clearInterval(interval);
  }, []);
  
  const triggerDataCollection = async () => {
    const executionId = await n8nService.triggerDataCollection();
    // Add to executions list for tracking
    setExecutions(prev => [...prev, { id: executionId, status: 'running' }]);
    return executionId;
  };
  
  return {
    isHealthy,
    executions,
    triggerDataCollection
  };
};
```

---

## 🔧 **API TASARIMI**

### **RESTful API Endpoints (Updated)**
```typescript
// Vehicle Endpoints
GET    /api/vehicles                    // List vehicles with pagination
GET    /api/vehicles/:id                // Get vehicle details
GET    /api/vehicles/search             // Advanced search
GET    /api/vehicles/opportunities      // Today's opportunities
GET    /api/vehicles/city/:city         // Vehicles by city
GET    /api/vehicles/brand/:brand       // Vehicles by brand
GET    /api/vehicles/:id/packages       // Vehicle packages

// Price History Endpoints
GET    /api/vehicles/:id/price-history  // Vehicle price history
POST   /api/vehicles/:id/price-update   // Manual price update

// n8n Integration Endpoints
GET    /api/n8n/health                  // n8n health status
POST   /api/n8n/trigger/:workflow       // Trigger workflow
GET    /api/n8n/executions              // List executions
GET    /api/n8n/executions/:id          // Execution details

// Data Processing Endpoints
POST   /api/data/normalize               // Trigger normalization
GET    /api/data/batches                // List processing batches
GET    /api/data/batches/:id            // Batch details

// Webhook Endpoints (for n8n)
POST   /webhook/vehicle-updates         // Vehicle data updates
POST   /webhook/price-changes           // Price change notifications
POST   /webhook/system-health           // System health updates

// User Endpoints
POST   /api/auth/login                  // User login
POST   /api/auth/register               // User registration
GET    /api/user/profile                // User profile
PUT    /api/user/profile                // Update profile
GET    /api/user/favorites              // User favorites
POST   /api/user/favorites/:vehicleId   // Add to favorites
DELETE /api/user/favorites/:vehicleId   // Remove from favorites

// Analytics Endpoints
GET    /api/analytics/city-stats        // City statistics
GET    /api/analytics/brand-performance // Brand performance
GET    /api/analytics/market-trends     // Market trends
GET    /api/analytics/normalization     // Normalization statistics

// System Endpoints
GET    /api/health                      // Health check
GET    /api/status                      // System status
GET    /api/logs                        // System logs
```

---

**📅 Doküman Tarihi:** 2025-01-27  
**🏗️ Mimari Versiyon:** v2.0 (Updated with n8n v1.111.0)  
**🔄 Son Güncelleme:** n8n integration, normalization scripts, environment config  
**📊 Kapsam:** Full-stack system architecture with data processing pipeline  
**🎯 Durum:** Ready for implementation with enhanced n8n workflows