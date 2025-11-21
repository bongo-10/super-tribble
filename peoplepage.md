Here are the complete file modifications and additions required to implement the "Place of People (as provided)" page.

### 1. Update Elasticsearch Mapping
First, we need to ensure the address fields in the `details` object of the `people` index are explicitly mapped as keywords to support the necessary aggregations.

**File:** `backend_python/app/elasticsearch_client.py`

```python
# backend_python/app/elasticsearch_client.py
from elasticsearch import Elasticsearch, AsyncElasticsearch
from elasticsearch.helpers import bulk, async_bulk
from typing import Dict, Any, List, Optional
import logging
from .config import settings
import json

logger = logging.getLogger(__name__)

class ElasticsearchClient:
    """Elasticsearch client wrapper with connection management"""

    def __init__(self):
        self.client: Optional[Elasticsearch] = None
        self.async_client: Optional[AsyncElasticsearch] = None

    def connect(self):
        """Establish synchronous connection to Elasticsearch"""
        try:
            self.client = Elasticsearch(
                [settings.elasticsearch_url],
                basic_auth=(settings.elasticsearch_user, settings.elasticsearch_password),
                verify_certs=False,
                ssl_show_warn=False,
                request_timeout=30,
                max_retries=3,
                retry_on_timeout=True
            )
            if self.client.ping():
                logger.info(f"Connected to Elasticsearch at {settings.elasticsearch_url}")
            else:
                raise ConnectionError("Cannot connect to Elasticsearch")
        except Exception as e:
            logger.error(f"Elasticsearch connection error: {str(e)}")
            raise

    def connect_async(self):
        """Establish asynchronous connection to Elasticsearch"""
        try:
            self.async_client = AsyncElasticsearch(
                [settings.elasticsearch_url],
                basic_auth=(settings.elasticsearch_user, settings.elasticsearch_password),
                verify_certs=False,
                ssl_show_warn=False,
                request_timeout=30,
                max_retries=3,
                retry_on_timeout=True
            )
            logger.info(f"Async client configured for Elasticsearch at {settings.elasticsearch_url}")
        except Exception as e:
            logger.error(f"Elasticsearch async connection error: {str(e)}")
            raise

    def close(self):
        if self.client:
            self.client.close()
            logger.info("Elasticsearch connection closed")

    async def close_async(self):
        if self.async_client:
            await self.async_client.close()
            logger.info("Elasticsearch async connection closed")

    def create_indices(self):
        """Create Elasticsearch indices with mappings"""
        if not self.client:
            raise ConnectionError("Synchronous client not connected.")

        # --- Define reusable property mappings ---
        applicant_properties = {
            "emailAddress": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
            "mobilePhoneNumber": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
            "mobilePhoneNumber_normalized": {"type": "keyword"},
            "approvalStatus": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
            "nationalID": {"type": "keyword"},
            "passportNumber": {"type": "keyword"},
            "nida_region": {"type": "keyword"},
            "nida_district": {"type": "keyword"},
            "nida_ward": {"type": "keyword"}
        }

        person_nested_properties = {
            "emailAddress": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
            "mobilePhoneNumber": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
            "mobilePhoneNumber_normalized": {"type": "keyword"},
            "nationalId": {"type": "keyword"},
            "nationalID": {"type": "keyword"},
            "passportNumber": {"type": "keyword"},
            "shareholderId": {"type": "keyword"},
            "type": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
            "country": {"type": "keyword"},
            "lastName": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
            "firstName": {"type": "text"},
            "middleName": {"type": "text"},
            "dateOfBirth": {"type": "keyword"},
            "nationality": {"type": "keyword"},
            # NIDA Location
            "nida_region": {"type": "keyword"},
            "nida_district": {"type": "keyword"},
            "nida_ward": {"type": "keyword"},
            # Provided Address (Added for Place of People Provided)
            "region": {"type": "keyword"},
            "district": {"type": "keyword"},
            "ward": {"type": "keyword"},
            "street": {"type": "keyword"},
            "road": {"type": "keyword"},
            "typeOfLocalAddress": {"type": "keyword"},
            # Financials
            "shares_count": {"type": "long"},
            "share_value": {"type": "double"}
        }

        autocomplete_analyzer = {
            "analysis": {
                "filter": {
                    "autocomplete_filter": {
                        "type": "edge_ngram",
                        "min_gram": 2,
                        "max_gram": 20
                    }
                },
                "analyzer": {
                    "autocomplete_analyzer": {
                        "type": "custom",
                        "tokenizer": "standard",
                        "filter": [
                            "lowercase",
                            "autocomplete_filter"
                        ]
                    }
                }
            }
        }

        business_place_properties = {
            "type": "object",
            "properties": {
                "region": {"type": "keyword"},
                "district": {"type": "keyword"},
                "ward": {"type": "keyword"},
                "streetBusiness": {"type": "keyword"},
                "roadBusiness": {"type": "keyword"},
                "typeOfLocalAddress": {"type": "keyword"},
                # New Fields for Contact Info
                "mobilePhoneNumberCmpny": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                "mobilePhoneNumberBsness": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                "emailAddressCmpny": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                "emailAddressBsness": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                # Helper fields for phone filter logic
                "mobilePhoneNumberCmpny_normalized": {"type": "keyword"},
                "mobilePhoneNumberBsness_normalized": {"type": "keyword"}
            }
        }

        # Shared display_name for cross-index sorting
        display_name_mapping = {
            "display_name": {"type": "keyword"}
        }

        # --- Company Index Mapping ---
        companies_mapping = {
            "mappings": {
                "properties": {
                    "id": {"type": "keyword"},
                    "company_name": {
                        "type": "text",
                        "fields": {
                            "keyword": {"type": "keyword"},
                            "suggest": {"type": "completion"},
                            "autocomplete": {"type": "text", "analyzer": "autocomplete_analyzer", "search_analyzer": "standard"}
                        },
                        "copy_to": "display_name"
                    },
                    **display_name_mapping,
                    "registration_type": {"type": "keyword"},
                    "incorporation_number": {"type": "keyword"},
                    "tin": {"type": "keyword"},
                    "incorporation_date": {"type": "keyword"},
                    "applicant_info": {"type": "object", "properties": applicant_properties},
                    "business_place": business_place_properties,
                    "company_secretary_detail": {"type": "object", "properties": person_nested_properties},
                    "other_info": {"type": "object", "enabled": False},
                    "director_details": {"type": "nested", "properties": person_nested_properties},
                    "shareholder_details": {"type": "nested", "properties": person_nested_properties},
                    "business_activities": {"type": "keyword"},
                    "created_at": {"type": "date"},
                    "authorised_share": {"type": "long"},
                    "time_application_created": {"type": "keyword"},
                    "time_application_submitted": {"type": "keyword"},
                    "service_type": {"type": "keyword"}
                }
            },
            "settings": {
                "number_of_shards": 1,
                "number_of_replicas": 1,
                "index.mapping.total_fields.limit": 2000,
                "index.max_result_window": 3000000,
                **autocomplete_analyzer
            }
        }

        # --- Business Name Index Mapping ---
        business_names_mapping = {
            "mappings": {
                "properties": {
                    "id": {"type": "keyword"},
                    "business_name": {
                        "type": "text",
                        "fields": {
                            "keyword": {"type": "keyword"},
                            "suggest": {"type": "completion"},
                            "autocomplete": {"type": "text", "analyzer": "autocomplete_analyzer", "search_analyzer": "standard"}
                        },
                        "copy_to": "display_name"
                    },
                    **display_name_mapping,
                    "registration_type": {"type": "keyword"},
                    "registration_number": {"type": "keyword"},
                    "registration_date": {"type": "keyword"},
                    "business_type": {"type": "keyword"},
                    "applicant_info": {"type": "object", "properties": applicant_properties},
                    "business_place": business_place_properties,
                    "owner_details": {"type": "nested", "properties": person_nested_properties},
                    "persons_who_can_update_details": {"type": "nested", "properties": person_nested_properties},
                    "bank_operator_details": {"type": "nested", "properties": person_nested_properties},
                    "business_activities": {"type": "keyword"},
                    "created_at": {"type": "date"},
                    "time_application_created": {"type": "keyword"}
                }
            },
            "settings": {
                "number_of_shards": 1,
                "number_of_replicas": 1,
                "index.mapping.total_fields.limit": 2000,
                "index.max_result_window": 3000000,
                **autocomplete_analyzer
            }
        }

        # --- People Index Mapping (Unified) ---
        people_mapping = {
            "mappings": {
                "properties": {
                    "person_identifier": {"type": "keyword"},
                    "full_name": {
                        "type": "text",
                        "fields": {
                            "keyword": {"type": "keyword"},
                            "suggest": {"type": "completion"},
                            "autocomplete": {"type": "text", "analyzer": "autocomplete_analyzer", "search_analyzer": "standard"}
                        },
                        "copy_to": "display_name"
                    },
                    **display_name_mapping,
                    "date_of_birth": {"type": "date", "format": "yyyy-MM-dd||dd/MM/yyyy||epoch_millis"},
                    "nationality": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                    "role": {"type": "keyword"},
                    "registration_id": {"type": "keyword"},
                    "source_record_id": {"type": "keyword"},
                    "registration_name": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                    "registration_type": {"type": "keyword"},
                    "approval_status": {"type": "keyword"},
                    "record_created_at": {"type": "date", "format": "dd/MM/yyyy HH:mm:ss"},
                    "details": {"type": "object", "properties": person_nested_properties}
                }
            },
            "settings": {
                "number_of_shards": 1,
                "number_of_replicas": 1,
                "index.mapping.total_fields.limit": 2000,
                "index.max_result_window": 3000000,
                **autocomplete_analyzer
            }
        }

        # --- NEW: Corporate Shareholder Index Mapping ---
        corporate_shareholders_mapping = {
            "mappings": {
                "properties": {
                    "id": {"type": "keyword"},
                    "name": {"type": "text", "fields": {"keyword": {"type": "keyword"}, "autocomplete": {"type": "text", "analyzer": "autocomplete_analyzer", "search_analyzer": "standard"}}},
                    "country": {"type": "keyword"},
                    "email_address": {"type": "keyword"},
                    "phone_number": {"type": "keyword"},
                    "phone_number_normalized": {"type": "keyword"},
                    "shareholdings": {
                        "type": "nested",
                        "properties": {
                            "company_id": {"type": "keyword"},
                            "company_name": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                            "shares_class": {"type": "keyword"},
                            "shares_count": {"type": "long"}
                        }
                    },
                    "shareholding_count": {"type": "integer"},
                    "created_at": {"type": "date"}
                }
            },
            "settings": {
                "number_of_shards": 1,
                "number_of_replicas": 1,
                "index.mapping.total_fields.limit": 2000,
                "index.max_result_window": 3000000,
                **autocomplete_analyzer
            }
        }

        # --- NEW: TRA TINS Index Mapping ---
        tra_tins_mapping = {
            "mappings": {
                "properties": {
                    "id": {"type": "keyword"},
                    "tin_no": {"type": "keyword"},
                    "name": {
                        "type": "text",
                        "fields": {
                            "keyword": {"type": "keyword"},
                            "autocomplete": {"type": "text", "analyzer": "autocomplete_analyzer", "search_analyzer": "standard"}
                        }
                    },
                    "firstname": {"type": "text"},
                    "middlename": {"type": "text"},
                    "lastname": {"type": "text"},
                    "tin_type": {"type": "keyword"},
                    "national_id": {"type": "keyword"},
                    "incorporation_number": {"type": "keyword"},
                    "telephones": {"type": "keyword"},
                    "telephones_normalized": {"type": "keyword"},
                    "business_start_date": {"type": "date", "format": "M/d/yyyy H:mm:ss a||yyyy-MM-dd||epoch_millis", "ignore_malformed": True},
                    "pobox": {"type": "keyword"},
                    "postal_city": {"type": "keyword"},
                    "created_at": {"type": "date"},
                    "date_of_birth": {"type": "date", "format": "yyyy-MM-dd", "ignore_malformed": True},
                    "age": {"type": "integer"},
                    "nida_region": {"type": "keyword"},
                    "nida_district": {"type": "keyword"},
                    "nida_ward": {"type": "keyword"}
                }
            },
            "settings": {
                "number_of_shards": 1,
                "number_of_replicas": 1,
                "index.max_result_window": 3000000,
                **autocomplete_analyzer
            }
        }

        # --- NEW: Wabunge Index Mapping ---
        wabunge_mapping = {
            "mappings": {
                "properties": {
                    "id": {"type": "keyword"},
                    "name": {
                        "type": "text",
                        "fields": {
                            "keyword": {"type": "keyword"},
                            "autocomplete": {"type": "text", "analyzer": "autocomplete_analyzer", "search_analyzer": "standard"}
                        }
                    },
                    "title": {"type": "keyword"},
                    "phone1": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                    "phone1_normalized": {"type": "keyword"},
                    "phone2": {"type": "text", "fields": {"keyword": {"type": "keyword"}}},
                    "phone2_normalized": {"type": "keyword"},
                    "chama": {"type": "keyword"},
                    "address1": {"type": "text"},
                    "address2": {"type": "text"},
                    "created_at": {"type": "date"}
                }
            },
            "settings": {
                "number_of_shards": 1,
                "number_of_replicas": 1,
                "index.max_result_window": 10000,
                **autocomplete_analyzer
            }
        }

        indices_to_create = {
            "companies": companies_mapping,
            "business_names": business_names_mapping,
            "people": people_mapping,
            "corporate_shareholders": corporate_shareholders_mapping,
            "tra_tins": tra_tins_mapping,
            "wabunge": wabunge_mapping
        }

        for index_name, mapping in indices_to_create.items():
            if self.client.indices.exists(index=index_name):
                self.client.indices.delete(index=index_name)
                logger.info(f"Deleted existing '{index_name}' index.")
            
            self.client.indices.create(index=index_name, body=mapping)
            logger.info(f"Created '{index_name}' index")

es_client = ElasticsearchClient()
```

### 2. Define Backend Models
We need a new response model for the provided places.

**File:** `backend_python/app/models/places.py`

```python
# backend_python/app/models/places.py
from pydantic import BaseModel, Field
from typing import List, Dict, Any
from .shared import PaginationResponse

class PlaceItem(BaseModel):
    id: str
    place_name: str = Field(..., alias="placeName")
    place_type: str = Field(..., alias="placeType")
    count: int

class PlacesListResponse(BaseModel):
    success: bool = True
    places: List[PlaceItem]
    pagination: Dict[str, Any]

class PeoplePlaceItem(BaseModel):
    id: str # Encoded key for consistent ID referencing if needed
    region: str
    district: str
    ward: str
    place_name: str = Field(..., alias="placeName")
    count: int

class PeoplePlacesListResponse(BaseModel):
    success: bool = True
    places: List[PeoplePlaceItem]
    pagination: PaginationResponse

# NEW: For the "As Provided" functionality
class PeoplePlaceProvidedItem(BaseModel):
    id: str
    region: str
    district: str
    ward: str
    street: str
    road: str
    place_name: str = Field(..., alias="placeName")
    address_type: str = Field(..., alias="addressType")
    count: int

class PeoplePlacesProvidedListResponse(BaseModel):
    success: bool = True
    places: List[PeoplePlaceProvidedItem]
    pagination: PaginationResponse
```

### 3. Backend Route for Places
Implement the new endpoint that performs the composite aggregation including street and road.

**File:** `backend_python/app/routes/places.py`

```python
# backend_python/app/routes/places.py
from fastapi import APIRouter, Query, HTTPException
from typing import Optional, List, Dict
import logging
import json
import base64
from ..models import PlacesListResponse, PeoplePlacesListResponse, PeoplePlacesProvidedListResponse, ErrorResponse
from ..elasticsearch_client import es_client
from ..utils.helpers import transform_company_for_list, transform_business_name_for_list

router = APIRouter(prefix="/api", tags=["places"])
logger = logging.getLogger(__name__)

def encode_place_id(place_key: Dict) -> str:
    """Encodes the composite aggregation key into a URL-safe string."""
    try:
        json_str = json.dumps(place_key, sort_keys=True)
        return base64.urlsafe_b64encode(json_str.encode()).decode()
    except Exception as e:
        logger.error(f"Error encoding place_id: {e}")
        return ""

def decode_place_id(encoded: str) -> Optional[Dict]:
    """Decodes the URL-safe string back into a dictionary key."""
    try:
        if not encoded:
            return None
        json_str = base64.urlsafe_b64decode(encoded.encode()).decode()
        return json.loads(json_str)
    except Exception as e:
        logger.error(f"Error decoding place_id: {e}")
        return None

@router.get("/places", response_model=PlacesListResponse)
async def list_places_of_business(
    registration_type: str = Query(..., description="Type of registration: 'company' or 'business_name'"),
    search: Optional[str] = Query(None),
    approvedOnly: bool = Query(False),
    page: int = Query(1, ge=1),
    limit: int = Query(25, ge=1, le=100),
):
    try:
        index_name = "companies" if registration_type == "company" else "business_names"
        
        query_must = []
        if approvedOnly:
            query_must.append({"term": {"applicant_info.approvalStatus.keyword": "Approve"}})
        
        # The ES query for search will run on the granular fields first
        if search:
            query_must.append({
                "multi_match": {
                    "query": search,
                    "fields": [
                        "business_place.region", "business_place.district", 
                        "business_place.ward", "business_place.streetBusiness",
                        "business_place.roadBusiness"
                    ],
                    "fuzziness": "AUTO"
                }
            })

        query = {"bool": {"must": query_must}} if query_must else {"match_all": {}}

        # Step 1: Fetch all granular combinations from Elasticsearch using composite aggregation
        aggs = {
            "places": {
                "composite": {
                    "size": 10000,
                    "sources": [
                        {"region": {"terms": {"field": "business_place.region", "missing_bucket": True}}},
                        {"district": {"terms": {"field": "business_place.district", "missing_bucket": True}}},
                        {"ward": {"terms": {"field": "business_place.ward", "missing_bucket": True}}},
                        {"street": {"terms": {"field": "business_place.streetBusiness", "missing_bucket": True}}},
                        {"road": {"terms": {"field": "business_place.roadBusiness", "missing_bucket": True}}},
                        {"type": {"terms": {"field": "business_place.typeOfLocalAddress", "missing_bucket": True}}}
                    ]
                }
            }
        }

        result = await es_client.async_client.search(index=index_name, query=query, size=0, aggs=aggs, request_timeout=60)

        # Step 2: Aggregate the results in memory to group by the desired display fields
        aggregated_places: Dict[str, Dict] = {}
        
        for bucket in result["aggregations"]["places"]["buckets"]:
            key = bucket["key"]
            # Create a simplified display name and a key for aggregation
            place_parts = [key.get("region"), key.get("district"), key.get("ward"), key.get("street"), key.get("road")]
            display_name = ", ".join(filter(None, place_parts))
            
            # Skip if the place name is essentially empty
            if not display_name.strip():
                continue
                
            aggregation_key = display_name
            
            if aggregation_key not in aggregated_places:
                # Create a representative place_id for the link on the detail page.
                # This ensures the detail page fetches all records for this aggregated place.
                representative_place_dict = {
                    "region": key.get("region"),
                    "district": key.get("district"),
                    "ward": key.get("ward"),
                    "street": key.get("street"),
                    "road": key.get("road"),
                }
                aggregated_places[aggregation_key] = {
                    "id": encode_place_id(representative_place_dict),
                    "placeName": display_name,
                    "placeType": key.get("type") or "N/A",
                    "count": 0
                }
            
            aggregated_places[aggregation_key]["count"] += bucket["doc_count"]

        # Convert the aggregated dictionary to a list
        all_places = list(aggregated_places.values())
        
        # Sort the final aggregated list by count
        all_places.sort(key=lambda x: x['count'], reverse=True)

        # Step 3: Paginate the final, correctly counted list
        total_count = len(all_places)
        total_pages = (total_count + limit - 1) // limit
        start_idx = (page - 1) * limit
        end_idx = start_idx + limit
        paginated_places = all_places[start_idx:end_idx]

        return {
            "success": True,
            "places": paginated_places,
            "pagination": {
                "currentPage": page,
                "totalPages": total_pages,
                "totalCount": total_count,
                "limit": limit,
                "searchAfter": None,
                "hasMore": page < total_pages
            }
        }

    except Exception as e:
        logger.error(f"Error fetching places of business: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/registrations-by-place")
async def list_registrations_by_place(
    registration_type: str = Query(..., description="Type of registration: 'company' or 'business_name'"),
    place_id: str = Query(..., description="Encoded ID of the place"),
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100),
    approvedOnly: bool = Query(False)
):
    try:
        place_key = decode_place_id(place_id)
        if not place_key:
            raise HTTPException(status_code=400, detail="Invalid place_id")

        index_name = "companies" if registration_type == "company" else "business_names"
        
        must_conditions = []
        if approvedOnly:
            must_conditions.append({"term": {"applicant_info.approvalStatus.keyword": "Approve"}})

        # Map API keys to ES field names
        key_to_field_map = {
            "region": "business_place.region",
            "district": "business_place.district",
            "ward": "business_place.ward",
            "street": "business_place.streetBusiness",
            "road": "business_place.roadBusiness",
        }

        for key, value in place_key.items():
            field_name = key_to_field_map.get(key)
            if field_name:
                if value is None:
                    must_conditions.append({"bool": {"must_not": [{"exists": {"field": field_name}}]}})
                else:
                    must_conditions.append({"term": {field_name: value}})

        query = {"bool": {"must": must_conditions}}
        
        search_params = {
            "index": index_name,
            "query": query,
            "size": limit,
            "from_": (page - 1) * limit,
            "sort": [{"created_at": "desc"}],
            "track_total_hits": True
        }

        result = await es_client.async_client.search(**search_params)
        
        transform_func = transform_company_for_list if registration_type == "company" else transform_business_name_for_list
        registrations = [transform_func(hit) for hit in result["hits"]["hits"]]
        
        total_count = result["hits"]["total"]["value"]
        total_pages = (total_count + limit - 1) // limit

        return {
            "success": True,
            "registrations": registrations,
            "pagination": {
                "currentPage": page,
                "totalPages": total_pages,
                "totalCount": total_count,
                "limit": limit,
                "hasMore": page < total_pages
            }
        }

    except Exception as e:
        logger.error(f"Error fetching registrations by place: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))

# --- List Places of People (NIDA Location) ---
@router.get("/people-places", response_model=PeoplePlacesListResponse)
async def list_people_places(
    registration_type: str = Query(..., description="Type of registration: 'company' or 'business_name'"),
    search: Optional[str] = Query(None),
    approvedOnly: bool = Query(False),
    page: int = Query(1, ge=1),
    limit: int = Query(25, ge=1, le=100),
):
    try:
        # We are querying the 'people' index
        # First, filter by registration_type
        reg_type_param = "company" if registration_type == "company" else "business-name"
        
        query_must = [
            {"term": {"registration_type": reg_type_param}}
        ]
        
        if approvedOnly:
            query_must.append({"term": {"approval_status": "Approve"}})

        # Search clause targeting NIDA fields
        if search:
            query_must.append({
                "multi_match": {
                    "query": search,
                    "fields": [
                        "details.nida_region", "details.nida_district", "details.nida_ward"
                    ],
                    "fuzziness": "AUTO"
                }
            })

        # Ensure we only aggregate records that actually have NIDA location data
        query_must.append({
            "bool": {
                "should": [
                    {"exists": {"field": "details.nida_region"}},
                    {"exists": {"field": "details.nida_district"}},
                    {"exists": {"field": "details.nida_ward"}}
                ],
                "minimum_should_match": 1
            }
        })

        query = {"bool": {"must": query_must}}

        # Composite aggregation to group by Region, District, Ward
        aggs = {
            "nida_places": {
                "composite": {
                    "size": 10000, # Large enough to fetch most places for in-memory sorting
                    "sources": [
                        {"region": {"terms": {"field": "details.nida_region", "missing_bucket": False}}},
                        {"district": {"terms": {"field": "details.nida_district", "missing_bucket": False}}},
                        {"ward": {"terms": {"field": "details.nida_ward", "missing_bucket": False}}}
                    ]
                }
            }
        }

        result = await es_client.async_client.search(index="people", query=query, size=0, aggs=aggs, request_timeout=60)

        aggregated_places = []
        for bucket in result["aggregations"]["nida_places"]["buckets"]:
            key = bucket["key"]
            region = key.get("region")
            district = key.get("district")
            ward = key.get("ward")
            
            # Construct display name
            parts = [p for p in [region, district, ward] if p]
            display_name = ", ".join(parts)
            
            if not display_name.strip():
                continue

            aggregated_places.append({
                "id": "N/A", # ID not strictly used for navigation here, we use query params
                "region": region or "",
                "district": district or "",
                "ward": ward or "",
                "placeName": display_name,
                "count": bucket["doc_count"]
            })

        # Sort by count descending
        aggregated_places.sort(key=lambda x: x['count'], reverse=True)

        # Pagination
        total_count = len(aggregated_places)
        total_pages = (total_count + limit - 1) // limit
        start_idx = (page - 1) * limit
        end_idx = start_idx + limit
        paginated_places = aggregated_places[start_idx:end_idx]

        return {
            "success": True,
            "places": paginated_places,
            "pagination": {
                "currentPage": page,
                "totalPages": total_pages,
                "totalCount": total_count,
                "limit": limit,
                "searchAfter": None,
                "hasMore": page < total_pages
            }
        }

    except Exception as e:
        logger.error(f"Error fetching places of people: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))

# --- NEW: List Places of People (As Provided) ---
@router.get("/people-places-provided", response_model=PeoplePlacesProvidedListResponse)
async def list_people_places_provided(
    registration_type: str = Query(..., description="Type of registration: 'company' or 'business_name'"),
    search: Optional[str] = Query(None),
    approvedOnly: bool = Query(False),
    page: int = Query(1, ge=1),
    limit: int = Query(25, ge=1, le=100),
):
    try:
        # Query the 'people' index
        reg_type_param = "company" if registration_type == "company" else "business-name"
        
        query_must = [
            {"term": {"registration_type": reg_type_param}}
        ]
        
        if approvedOnly:
            query_must.append({"term": {"approval_status": "Approve"}})

        # Search across all provided location fields
        if search:
            query_must.append({
                "multi_match": {
                    "query": search,
                    "fields": [
                        "details.region", "details.district", "details.ward",
                        "details.street", "details.road"
                    ],
                    "fuzziness": "AUTO"
                }
            })

        # Only consider records that have at least ONE of the relevant location fields
        query_must.append({
            "bool": {
                "should": [
                    {"exists": {"field": "details.region"}},
                    {"exists": {"field": "details.district"}},
                    {"exists": {"field": "details.ward"}},
                    {"exists": {"field": "details.street"}},
                    {"exists": {"field": "details.road"}}
                ],
                "minimum_should_match": 1
            }
        })

        query = {"bool": {"must": query_must}}

        # Composite aggregation for all provided fields
        # missing_bucket: True ensures we don't skip records where a specific field is null
        aggs = {
            "provided_places": {
                "composite": {
                    "size": 10000, 
                    "sources": [
                        {"region": {"terms": {"field": "details.region", "missing_bucket": True}}},
                        {"district": {"terms": {"field": "details.district", "missing_bucket": True}}},
                        {"ward": {"terms": {"field": "details.ward", "missing_bucket": True}}},
                        {"street": {"terms": {"field": "details.street", "missing_bucket": True}}},
                        {"road": {"terms": {"field": "details.road", "missing_bucket": True}}},
                        {"type": {"terms": {"field": "details.typeOfLocalAddress", "missing_bucket": True}}}
                    ]
                }
            }
        }

        result = await es_client.async_client.search(index="people", query=query, size=0, aggs=aggs, request_timeout=60)

        aggregated_places = []
        for bucket in result["aggregations"]["provided_places"]["buckets"]:
            key = bucket["key"]
            
            # Clean up fields (replace nulls/None with empty string for display logic)
            region = key.get("region") or ""
            district = key.get("district") or ""
            ward = key.get("ward") or ""
            street = key.get("street") or ""
            road = key.get("road") or ""
            address_type = key.get("type") or "N/A"

            # Construct Display Name
            parts = [p for p in [region, district, ward, street, road] if p]
            display_name = ", ".join(parts)

            if not display_name.strip():
                continue

            aggregated_places.append({
                "id": "N/A", 
                "region": region,
                "district": district,
                "ward": ward,
                "street": street,
                "road": road,
                "placeName": display_name,
                "addressType": address_type,
                "count": bucket["doc_count"]
            })

        # Sort by count descending
        aggregated_places.sort(key=lambda x: x['count'], reverse=True)

        # Pagination
        total_count = len(aggregated_places)
        total_pages = (total_count + limit - 1) // limit
        start_idx = (page - 1) * limit
        end_idx = start_idx + limit
        paginated_places = aggregated_places[start_idx:end_idx]

        return {
            "success": True,
            "places": paginated_places,
            "pagination": {
                "currentPage": page,
                "totalPages": total_pages,
                "totalCount": total_count,
                "limit": limit,
                "searchAfter": None,
                "hasMore": page < total_pages
            }
        }

    except Exception as e:
        logger.error(f"Error fetching provided places of people: {e}", exc_info=True)
        raise HTTPException(status_code=500, detail=str(e))
```

### 4. Backend People Route Update
Update `list_people` to accept the new location query parameters.

**File:** `backend_python/app/routes/people.py`

```python
# backend_python/app/routes/people.py
from fastapi import APIRouter, Query, HTTPException
from typing import Optional, List
import logging
import json
import base64
from ..models import PeopleListResponse, ErrorResponse
from ..elasticsearch_client import es_client
from ..utils.helpers import build_es_query, calculate_age_range, transform_person_for_list

router = APIRouter(prefix="/api", tags=["people"])
logger = logging.getLogger(__name__)

def encode_search_after(sort_values: List) -> str:
    try:
        json_str = json.dumps(sort_values)
        return base64.b64encode(json_str.encode()).decode()
    except Exception as e:
        logger.error(f"Error encoding search_after: {e}")
        return ""

def decode_search_after(encoded: str) -> Optional[List]:
    try:
        if not encoded:
            return None
        json_str = base64.b64decode(encoded.encode()).decode()
        return json.loads(json_str)
    except Exception as e:
        logger.error(f"Error decoding search_after: {e}")
        return None

@router.get("/people", response_model=PeopleListResponse, responses={500: {"model": ErrorResponse}})
async def list_people(
    page: int = Query(1, ge=1, description="Page number"),
    limit: int = Query(25, ge=1, le=100, description="Items per page"),
    registration_type: str = Query('company', description="Type of registration: 'company' or 'business_name'"),
    search: Optional[str] = Query(None, description="Search by name"),
    wholeWord: bool = Query(False),
    wholeSentence: bool = Query(False),
    role: Optional[str] = Query(None, description="Filter by role"),
    nationality: Optional[str] = Query(None, description="Filter by nationality"),
    minAge: Optional[int] = Query(None, ge=0),
    maxAge: Optional[int] = Query(None, ge=0),
    # NIDA Filters
    nidaRegion: Optional[str] = Query(None, description="Filter by NIDA Region"),
    nidaDistrict: Optional[str] = Query(None, description="Filter by NIDA District"),
    nidaWard: Optional[str] = Query(None, description="Filter by NIDA Ward"),
    # Provided Address Filters
    addressRegion: Optional[str] = Query(None, description="Filter by Provided Region"),
    addressDistrict: Optional[str] = Query(None, description="Filter by Provided District"),
    addressWard: Optional[str] = Query(None, description="Filter by Provided Ward"),
    addressStreet: Optional[str] = Query(None, description="Filter by Provided Street"),
    addressRoad: Optional[str] = Query(None, description="Filter by Provided Road"),
    
    approvedOnly: bool = Query(False, description="Filter for approved status only"),
    sortBy: str = Query("name", description="Sort field"),
    sortOrder: str = Query("asc", description="Sort order"),
    searchAfter: Optional[str] = Query(None, description="Search after cursor for pagination")
):
    try:
        # Handle mismatch between frontend param "business-name" and ES internal storage if any
        reg_type_param = "business-name" if registration_type == "business-name" else registration_type
        
        filter_conditions = [
            {"term": {"registration_type": reg_type_param}}
        ]

        if role:
            filter_conditions.append({"term": {"role": role}})
        if nationality:
            filter_conditions.append({"match": {"nationality": {"query": nationality, "fuzziness": "AUTO"}}})
        
        if approvedOnly:
            filter_conditions.append({"term": {"approval_status": "Approve"}})

        # --- NIDA Location Filters ---
        if nidaRegion:
            filter_conditions.append({"term": {"details.nida_region": nidaRegion}})
        if nidaDistrict:
            filter_conditions.append({"term": {"details.nida_district": nidaDistrict}})
        if nidaWard:
            filter_conditions.append({"term": {"details.nida_ward": nidaWard}})

        # --- NEW: Provided Address Filters ---
        # Using keyword fields for exact filtering from aggregation clicks
        if addressRegion:
            filter_conditions.append({"term": {"details.region": addressRegion}})
        if addressDistrict:
            filter_conditions.append({"term": {"details.district": addressDistrict}})
        if addressWard:
            filter_conditions.append({"term": {"details.ward": addressWard}})
        if addressStreet:
            filter_conditions.append({"term": {"details.street": addressStreet}})
        if addressRoad:
            filter_conditions.append({"term": {"details.road": addressRoad}})

        age_range = calculate_age_range(minAge, maxAge)
        if age_range:
            filter_conditions.append({"range": {"date_of_birth": age_range}})

        query = build_es_query(
            must_conditions=filter_conditions,
            search_term=search,
            search_fields=["full_name^3"],
            whole_word=wholeWord,
            whole_sentence=wholeSentence
        )

        sort_map = {
            "name": "full_name.keyword",
            "company": "registration_name.keyword",
            "dob": "date_of_birth"
        }
        sort_field = sort_map.get(sortBy, "full_name.keyword")
        sort_order_val = sortOrder.lower()
        
        sort_config = [
            {sort_field: {"order": sort_order_val}},
            {"person_identifier": {"order": "asc"}}
        ]

        use_search_after = page > 1 and searchAfter
        search_params = {
            "index": "people",
            "query": query,
            "size": limit,
            "sort": sort_config,
            "track_total_hits": True,
            "highlight": {"fields": {"full_name": {}}}
        }

        if use_search_after:
            decoded_search_after = decode_search_after(searchAfter)
            if decoded_search_after:
                search_params["search_after"] = decoded_search_after
        else:
            search_params["from_"] = (page - 1) * limit

        result = await es_client.async_client.search(**search_params)
        people_hits = result["hits"]["hits"]

        # --- START: Patch Nationality Logic ---
        person_ids_on_page = list(set([
            hit["_source"]["person_identifier"] for hit in people_hits 
            if hit.get("_source") and hit["_source"].get("person_identifier")
        ]))
        
        nationality_map = {}
        if person_ids_on_page:
            nationality_query = {
                "size": 0,
                "query": {"terms": {"person_identifier": person_ids_on_page}},
                "aggs": {
                    "people": {
                        "terms": {"field": "person_identifier", "size": len(person_ids_on_page)},
                        "aggs": {
                            "nationalities": {
                                "terms": {
                                    "field": "nationality.keyword",
                                    "size": 10 # Get top 10 distinct values
                                }
                            }
                        }
                    }
                }
            }
            nationality_res = await es_client.async_client.search(index="people", body=nationality_query)
            
            for bucket in nationality_res.get("aggregations", {}).get("people", {}).get("buckets", []):
                person_id = bucket.get("key")
                if not person_id: continue
                
                # Iterate through the returned nationality buckets to find the first valid one
                for nat_bucket in bucket.get("nationalities", {}).get("buckets", []):
                    nationality_val = nat_bucket.get("key")
                    # Robust check for a valid nationality string
                    if nationality_val and isinstance(nationality_val, str) and nationality_val.strip() and nationality_val != "noResult":
                        nationality_map[person_id] = nationality_val
                        break # Found a good one, move to the next person

        for hit in people_hits:
            source = hit.get("_source")
            if not source: continue
            
            current_nationality = source.get("nationality")
            if not current_nationality or current_nationality == "noResult" or not str(current_nationality).strip():
                person_id = source.get("person_identifier")
                if person_id and person_id in nationality_map:
                    source["nationality"] = nationality_map[person_id]
        # --- END: Patch Nationality Logic ---

        people = [transform_person_for_list(hit) for hit in people_hits]
        total_people = result["hits"]["total"]["value"]
        total_pages = (total_people + limit - 1) // limit
        next_search_after = None
        if people and len(result["hits"]["hits"]) > 0:
            last_hit = result["hits"]["hits"][-1]
            if "sort" in last_hit:
                next_search_after = encode_search_after(last_hit["sort"])

        return PeopleListResponse(
            success=True,
            people=people,
            pagination={
                "currentPage": page,
                "totalPages": total_pages,
                "totalCount": total_people,
                "limit": limit,
                "searchAfter": next_search_after,
                "hasMore": len(people) == limit
            }
        )

    except Exception as e:
        logger.error(f"Error fetching people: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))
```

### 5. Frontend Types
Update the TypeScript interfaces to include the new data type.

**File:** `src/types/places.ts`

```typescript
// src/types/places.ts

export interface Place {
    id: string; // base64 encoded key or simpler ID
    placeName: string;
    placeType: string;
    count: number;
}

export interface PeoplePlace {
    id: string;
    region: string;
    district: string;
    ward: string;
    placeName: string;
    count: number;
}

export interface PeoplePlaceProvided {
    id: string;
    region: string;
    district: string;
    ward: string;
    street: string;
    road: string;
    placeName: string;
    addressType: string;
    count: number;
}
```

### 6. Frontend Page: PlaceOfPeopleAsProvidedPage
Create the new React component.

**File:** `src/pages/PlaceOfPeopleAsProvidedPage.tsx`

```tsx
// leads_webapp1/src/pages/PlaceOfPeopleAsProvidedPage.tsx
import React, { useState, useEffect, useMemo } from 'react';
import { Link, useSearchParams } from 'react-router-dom';
import { AdvancedSearchBar, SearchParams } from '../components/AdvancedSearchBar';
import { Pagination } from '../components/Pagination';
import { RegistrationType, PeoplePlaceProvided } from '../types';

const API_URL = import.meta.env.VITE_API_BASE_URL;
const ITEMS_PER_PAGE = 25;

interface PaginationData {
    currentPage: number;
    totalPages: number;
    totalCount: number;
    limit: number;
}

interface PlaceOfPeopleAsProvidedPageProps {
    registrationType: RegistrationType;
}

const PlaceOfPeopleAsProvidedPage: React.FC<PlaceOfPeopleAsProvidedPageProps> = ({ registrationType }) => {
    const [searchParams, setSearchParams] = useSearchParams();
    const currentPage = parseInt(searchParams.get('page') || '1', 10);
    const approvedOnly = searchParams.get('approvedOnly') === 'true';

    const currentSearchParams: SearchParams = useMemo(() => ({
        term: searchParams.get('search') || '',
        wholeWord: false, 
        wholeSentence: false,
    }), [searchParams]);

    const [places, setPlaces] = useState<PeoplePlaceProvided[]>([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);
    const [paginationData, setPaginationData] = useState<PaginationData>({
        currentPage: 1,
        totalPages: 0,
        totalCount: 0,
        limit: ITEMS_PER_PAGE,
    });

    useEffect(() => {
        const fetchPlaces = async () => {
            setLoading(true);
            setError(null);
            try {
                const params = new URLSearchParams({
                    page: String(currentPage),
                    limit: String(ITEMS_PER_PAGE),
                    registration_type: registrationType,
                });

                if (currentSearchParams.term) {
                    params.append('search', currentSearchParams.term);
                }
                
                if (approvedOnly) {
                    params.append('approvedOnly', 'true');
                }

                const response = await fetch(`${API_URL}/api/people-places-provided?${params.toString()}`);
                if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
                
                const data = await response.json();
                if (!data.success) throw new Error(data.message || 'Failed to fetch places.');

                setPlaces(data.places);
                setPaginationData(data.pagination);
            } catch (e: any) {
                setError(e.message);
                console.error("Failed to fetch provided people places:", e);
            } finally {
                setLoading(false);
            }
        };

        fetchPlaces();
    }, [currentPage, currentSearchParams, registrationType, approvedOnly]);

    const handleSearchChange = (newSearchParams: SearchParams) => {
        setSearchParams(prev => {
            newSearchParams.term ? prev.set('search', newSearchParams.term) : prev.delete('search');
            prev.set('page', '1');
            return prev;
        }, { replace: true });
    };

    const handlePageChange = (newPage: number) => {
        if (newPage > 0 && newPage <= paginationData.totalPages && newPage !== currentPage) {
            setSearchParams(prev => {
                prev.set('page', String(newPage));
                return prev;
            });
            window.scrollTo(0, 0);
        }
    };

    const title = registrationType === 'company' ? 'Companies' : 'Business Names';

    const renderContent = () => {
        if (loading) {
            return <div className="text-center py-16"><h2 className="text-2xl font-semibold text-white">Loading Places...</h2></div>;
        }

        if (error) {
            return <div className="text-center py-16 px-4 bg-red-900 bg-opacity-30 rounded-lg"><h2 className="text-2xl font-semibold text-red-300">Failed to Load Data</h2><p className="text-red-400 mt-2">{error}</p></div>;
        }

        if (places.length > 0) {
            return (
                <>
                    <div className="bg-neutral rounded-lg shadow-lg border border-gray-800 overflow-x-auto">
                        <table className="w-full text-left text-sm">
                            <thead className="bg-gray-800 text-xs uppercase text-gray-400">
                                <tr>
                                    <th className="p-4">Place (Region, District, Ward, Street, Road)</th>
                                    <th className="p-4">Type</th>
                                    <th className="p-4 text-center">People Count</th>
                                    <th className="p-4 text-center">Actions</th>
                                </tr>
                            </thead>
                            <tbody className="text-gray-300">
                                {places.map((place, index) => {
                                    // Construct the query string for the PeoplePage filter
                                    const peopleQueryParams = new URLSearchParams();
                                    if (place.region) peopleQueryParams.set('addressRegion', place.region);
                                    if (place.district) peopleQueryParams.set('addressDistrict', place.district);
                                    if (place.ward) peopleQueryParams.set('addressWard', place.ward);
                                    if (place.street) peopleQueryParams.set('addressStreet', place.street);
                                    if (place.road) peopleQueryParams.set('addressRoad', place.road);
                                    
                                    if (approvedOnly) peopleQueryParams.set('approvedOnly', 'true');

                                    return (
                                        <tr key={index} className="border-b border-gray-800 hover:bg-gray-800/50 transition-colors">
                                            <td className="p-4 font-semibold">{place.placeName}</td>
                                            <td className="p-4">{place.addressType || 'N/A'}</td>
                                            <td className="p-4 text-center">
                                                <span className="px-3 py-1 text-xs font-semibold rounded-full bg-blue-800 text-blue-200">
                                                    {place.count.toLocaleString()}
                                                </span>
                                            </td>
                                            <td className="p-4 text-center">
                                                <Link 
                                                    to={`/${registrationType}/people?${peopleQueryParams.toString()}`}
                                                    className="inline-flex items-center gap-2 px-4 py-2 bg-accent hover:bg-blue-600 text-white rounded-md transition-colors font-semibold text-xs"
                                                >
                                                    View People
                                                </Link>
                                            </td>
                                        </tr>
                                    );
                                })}
                            </tbody>
                        </table>
                    </div>
                    <Pagination 
                        currentPage={paginationData.currentPage} 
                        totalPages={paginationData.totalPages} 
                        onPageChange={handlePageChange} 
                    />
                </>
            );
        }

        return (
            <div className="text-center py-16 px-4 bg-neutral rounded-lg">
                <h2 className="text-2xl font-semibold text-white">No Places Found</h2>
                <p className="text-gray-400 mt-2">
                    {currentSearchParams.term 
                        ? `Your search for "${currentSearchParams.term}" did not match any provided locations.`
                        : 'There are no places to display.'}
                </p>
            </div>
        );
    };

    return (
        <div className="space-y-8">
            <div>
                <h1 className="text-4xl font-bold text-white mb-2">Place of People (As Provided)</h1>
                <p className="text-lg text-gray-400">
                    Browse locations based on the address details provided by people associated with {title.toLowerCase()}.{' '}
                    {paginationData.totalCount > 0 ? `Found ${paginationData.totalCount.toLocaleString()} unique locations.` : ''}
                </p>
            </div>
            <AdvancedSearchBar 
                placeholder="Search by Region, District, Ward, Street..." 
                onSearchChange={handleSearchChange} 
                initialParams={currentSearchParams} 
            />
            {renderContent()}
        </div>
    );
};

export default PlaceOfPeopleAsProvidedPage;
```

### 7. Update Frontend PeoplePage
Modify to handle the new filtering parameters and display active filters.

**File:** `src/pages/PeoplePage.tsx`

```tsx
// src/pages/PeoplePage.tsx
import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { useSearchParams } from 'react-router-dom';
import { Person, RegistrationType } from '../types';
import { Pagination } from '../components/Pagination';
import { PeopleTable } from '../components/PeopleTable';
import { AdvancedSearchBar, SearchParams } from '../components/AdvancedSearchBar';

const useDebounce = (value: any, delay: number) => {
    const [debouncedValue, setDebouncedValue] = useState(value);
    useEffect(() => {
        const handler = setTimeout(() => {
            setDebouncedValue(value);
        }, delay);
        return () => {
            clearTimeout(handler);
        };
    }, [value, delay]);
    return debouncedValue;
};

const API_URL = import.meta.env.VITE_API_BASE_URL;
const PEOPLE_PER_PAGE = 25;

type SortKey = 'name' | 'company' | 'dob';
type SortOrder = 'asc' | 'desc';

interface PaginationData {
    currentPage: number;
    totalPages: number;
    totalCount: number;
}

interface PeoplePageProps {
    registrationType: RegistrationType;
}

const PeoplePage: React.FC<PeoplePageProps> = ({ registrationType }) => {
    const [searchParams, setSearchParams] = useSearchParams();

    // --- State derived from URL ---
    const currentPage = parseInt(searchParams.get('page') || '1', 10);
    const sortBy = (searchParams.get('sortBy') || 'name') as SortKey;
    const sortOrder = (searchParams.get('sortOrder') || 'asc') as SortOrder;

    const urlSearchParams = useMemo(() => ({
        search: searchParams.get('search') || '',
        wholeWord: searchParams.get('wholeWord') === 'true',
        wholeSentence: searchParams.get('wholeSentence') === 'true',
        role: searchParams.get('role') || '',
        nationality: searchParams.get('nationality') || '',
        minAge: searchParams.get('minAge') || '',
        maxAge: searchParams.get('maxAge') || '',
        approvedOnly: searchParams.get('approvedOnly') === 'true',
        // NIDA Filters
        nidaRegion: searchParams.get('nidaRegion') || '',
        nidaDistrict: searchParams.get('nidaDistrict') || '',
        nidaWard: searchParams.get('nidaWard') || '',
        // NEW: Provided Address Filters
        addressRegion: searchParams.get('addressRegion') || '',
        addressDistrict: searchParams.get('addressDistrict') || '',
        addressWard: searchParams.get('addressWard') || '',
        addressStreet: searchParams.get('addressStreet') || '',
        addressRoad: searchParams.get('addressRoad') || '',
    }), [searchParams]);

    // --- Local state for immediate UI feedback on filters ---
    const [roleFilter, setRoleFilter] = useState(urlSearchParams.role);
    const [nationalityFilter, setNationalityFilter] = useState(urlSearchParams.nationality);
    const [ageFilter, setAgeFilter] = useState({ min: urlSearchParams.minAge, max: urlSearchParams.maxAge });

    // Debounce filter inputs
    const debouncedNationality = useDebounce(nationalityFilter, 500);
    const debouncedAge = useDebounce(ageFilter, 500);

    // --- Component State ---
    const [people, setPeople] = useState<Person[]>([]);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState<string | null>(null);
    const [paginationData, setPaginationData] = useState<PaginationData>({
        currentPage: 1,
        totalPages: 0,
        totalCount: 0,
    });

    const title = registrationType === 'company' ? 'Company' : 'Business Name';

    // Effect to sync URL params to local state if URL changes from outside
    useEffect(() => {
        setRoleFilter(urlSearchParams.role);
        setNationalityFilter(urlSearchParams.nationality);
        setAgeFilter({ min: urlSearchParams.minAge, max: urlSearchParams.maxAge });
    }, [urlSearchParams]);

    // Effect to update URL from debounced local state
    useEffect(() => {
        const hasChanged = urlSearchParams.role !== roleFilter ||
            urlSearchParams.nationality !== debouncedNationality ||
            urlSearchParams.minAge !== debouncedAge.min ||
            urlSearchParams.maxAge !== debouncedAge.max;

        if (hasChanged) {
            setSearchParams(prev => {
                roleFilter ? prev.set('role', roleFilter) : prev.delete('role');
                debouncedNationality ? prev.set('nationality', debouncedNationality) : prev.delete('nationality');
                debouncedAge.min ? prev.set('minAge', debouncedAge.min) : prev.delete('minAge');
                debouncedAge.max ? prev.set('maxAge', debouncedAge.max) : prev.delete('maxAge');
                if (prev.get('page') !== '1') {
                    prev.set('page', '1');
                }
                return prev;
            }, { replace: true });
        }
    }, [roleFilter, debouncedNationality, debouncedAge, setSearchParams, urlSearchParams]);

    // --- Data Fetching Effect ---
    useEffect(() => {
        const fetchPeople = async () => {
            setLoading(true);
            setError(null);
            try {
                const params = new URLSearchParams({
                    page: String(currentPage),
                    limit: String(PEOPLE_PER_PAGE),
                    sortBy: sortBy,
                    sortOrder: sortOrder,
                    registration_type: registrationType,
                });

                if (urlSearchParams.search) {
                    params.append('search', urlSearchParams.search);
                    if (urlSearchParams.wholeWord) params.append('wholeWord', 'true');
                    if (urlSearchParams.wholeSentence) params.append('wholeSentence', 'true');
                }

                if (urlSearchParams.role) params.append('role', urlSearchParams.role);
                if (urlSearchParams.nationality) params.append('nationality', urlSearchParams.nationality);
                if (urlSearchParams.minAge) params.append('minAge', urlSearchParams.minAge);
                if (urlSearchParams.maxAge) params.append('maxAge', urlSearchParams.maxAge);
                if (urlSearchParams.approvedOnly) params.append('approvedOnly', 'true');

                // Pass NIDA params
                if (urlSearchParams.nidaRegion) params.append('nidaRegion', urlSearchParams.nidaRegion);
                if (urlSearchParams.nidaDistrict) params.append('nidaDistrict', urlSearchParams.nidaDistrict);
                if (urlSearchParams.nidaWard) params.append('nidaWard', urlSearchParams.nidaWard);

                // Pass Address params
                if (urlSearchParams.addressRegion) params.append('addressRegion', urlSearchParams.addressRegion);
                if (urlSearchParams.addressDistrict) params.append('addressDistrict', urlSearchParams.addressDistrict);
                if (urlSearchParams.addressWard) params.append('addressWard', urlSearchParams.addressWard);
                if (urlSearchParams.addressStreet) params.append('addressStreet', urlSearchParams.addressStreet);
                if (urlSearchParams.addressRoad) params.append('addressRoad', urlSearchParams.addressRoad);

                const response = await fetch(`${API_URL}/api/people?${params.toString()}`);
                if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
                const data = await response.json();
                if (!data.success) throw new Error(data.message || 'Failed to fetch people.');

                setPeople(data.people);
                setPaginationData(data.pagination);
            } catch (e: any) {
                setError(e.message);
                console.error("Failed to fetch people:", e);
            } finally {
                setLoading(false);
            }
        };

        fetchPeople();
    }, [currentPage, registrationType, urlSearchParams, sortBy, sortOrder]);

    const handleSearchChange = (newSearchParams: SearchParams) => {
        setSearchParams(prev => {
            newSearchParams.term ? prev.set('search', newSearchParams.term) : prev.delete('search');
            newSearchParams.wholeWord ? prev.set('wholeWord', 'true') : prev.delete('wholeWord');
            newSearchParams.wholeSentence ? prev.set('wholeSentence', 'true') : prev.delete('wholeSentence');
            prev.set('page', '1');
            return prev;
        }, { replace: true });
    };

    const handlePageChange = (newPage: number) => {
        if (newPage > 0 && newPage <= paginationData.totalPages && newPage !== currentPage) {
            setSearchParams(prev => {
                prev.set('page', String(newPage));
                return prev;
            });
            window.scrollTo(0, 0);
        }
    };

    const handleSort = useCallback((column: SortKey) => {
        const newSortOrder = sortBy === column && sortOrder === 'asc' ? 'desc' : 'asc';
        setSearchParams(prev => {
            prev.set('sortBy', column);
            prev.set('sortOrder', newSortOrder);
            prev.set('page', '1');
            return prev;
        }, { replace: true });
    }, [sortBy, sortOrder, setSearchParams]);

    // Construct display string for active NIDA filters
    const nidaFilters = [urlSearchParams.nidaRegion, urlSearchParams.nidaDistrict, urlSearchParams.nidaWard].filter(Boolean).join(' > ');
    
    // Construct display string for active Address filters
    const addressFilters = [
        urlSearchParams.addressRegion, 
        urlSearchParams.addressDistrict, 
        urlSearchParams.addressWard,
        urlSearchParams.addressStreet,
        urlSearchParams.addressRoad
    ].filter(Boolean).join(', ');

    if (error) {
        return (
            <div className="text-center py-16 px-4 bg-red-900 bg-opacity-30 rounded-lg">
                <h2 className="text-2xl font-semibold text-red-300">Failed to Load Data</h2>
                <p className="text-red-400 mt-2">{error}</p>
            </div>
        );
    }

    return (
        <div className="space-y-8">
            <div>
                <h1 className="text-4xl font-bold text-white mb-2">{title} People Directory</h1>
                <p className="text-lg text-gray-400">
                    Browse individuals associated with {title.toLowerCase()}s.{' '}
                    {paginationData.totalCount > 0 ? `Found ${paginationData.totalCount.toLocaleString()} records.` : ''}
                </p>
                {nidaFilters && (
                    <div className="mt-2 flex items-center gap-2">
                        <span className="text-sm font-bold text-slate-400">Filtering by NIDA Location:</span>
                        <span className="bg-blue-900 text-blue-200 px-3 py-1 rounded-full text-xs flex items-center gap-2">
                            {nidaFilters}
                            <button onClick={() => setSearchParams(prev => {
                                prev.delete('nidaRegion');
                                prev.delete('nidaDistrict');
                                prev.delete('nidaWard');
                                return prev;
                            })} className="hover:text-white font-bold"
                            >
                                &times;
                            </button>
                        </span>
                    </div>
                )}
                {addressFilters && (
                    <div className="mt-2 flex items-center gap-2">
                        <span className="text-sm font-bold text-slate-400">Filtering by Provided Address:</span>
                        <span className="bg-green-900 text-green-200 px-3 py-1 rounded-full text-xs flex items-center gap-2">
                            {addressFilters}
                            <button onClick={() => setSearchParams(prev => {
                                prev.delete('addressRegion');
                                prev.delete('addressDistrict');
                                prev.delete('addressWard');
                                prev.delete('addressStreet');
                                prev.delete('addressRoad');
                                return prev;
                            })} className="hover:text-white font-bold"
                            >
                                &times;
                            </button>
                        </span>
                    </div>
                )}
            </div>

            <AdvancedSearchBar 
                placeholder="Search by name..." 
                onSearchChange={handleSearchChange} 
                initialParams={{term: urlSearchParams.search, wholeWord: urlSearchParams.wholeWord, wholeSentence: urlSearchParams.wholeSentence}} 
            />

            {loading && people.length === 0 ? (
                <div className="text-center py-16 text-white text-xl">Loading People...</div>
            ) : (
                <div className={`${loading ? 'opacity-50' : ''} transition-opacity`}>
                    <PeopleTable 
                        people={people} 
                        roleFilter={roleFilter} 
                        onRoleChange={setRoleFilter}
                        nationalityFilter={nationalityFilter}
                        onNationalityChange={setNationalityFilter}
                        ageFilter={ageFilter}
                        onAgeChange={setAgeFilter}
                        sortBy={sortBy} 
                        sortOrder={sortOrder} 
                        onSort={handleSort}
                        registrationType={registrationType}
                    />
                    {paginationData.totalPages > 1 && (
                        <Pagination 
                            currentPage={paginationData.currentPage} 
                            totalPages={paginationData.totalPages} 
                            onPageChange={handlePageChange} 
                        />
                    )}
                </div>
            )}
        </div>
    );
};

export default PeoplePage;
```

### 8. Update Header
Add the new link under submenus.

**File:** `src/components/Header.tsx`

```tsx
// leads_webapp1/src/components/Header.tsx
import React from 'react';
import { Link, NavLink } from 'react-router-dom';
import { ApprovedOnlyToggle } from './ApprovedOnlyToggle';

interface HeaderProps {
    onToggleSidebar: () => void;
}

export const Header: React.FC<HeaderProps> = ({ onToggleSidebar }) => {
    const companyLinks = [
        { path: '/company/dashboard', label: 'Dashboard' },
        { path: '/company/people', label: 'People' },
        { path: '/company/activities', label: 'Activities' },
        { path: '/company/places', label: 'Place of Businesses' },
        { path: '/company/people-places', label: 'Place of People (NIDA)' },
        { path: '/company/people-places-provided', label: 'Place of People (As Provided)' }, // NEW
    ];

    const businessNameLinks = [
        { path: '/business-name/dashboard', label: 'Dashboard' },
        { path: '/business-name/people', label: 'People' },
        { path: '/business-name/activities', label: 'Activities' },
        { path: '/business-name/places', label: 'Place of Businesses' },
        { path: '/business-name/people-places', label: 'Place of People (NIDA)' },
        { path: '/business-name/people-places-provided', label: 'Place of People (As Provided)' }, // NEW
    ];

    const contactLinks = [
        { path: '/contacts', label: 'Contact Directory' },
        { path: '/phone-filter', label: 'Phone Filter' },
    ];

    const otherEntityLinks = [
        { path: '/corporate-shareholders', label: 'Corporate Shareholders' },
        { path: '/tra-tins', label: 'TRA Tins' },
        { path: '/wabunge', label: 'Wabunge' },
    ];

    const vizLinks = [
        { path: '/network-graph', label: 'Network Graph' },
        { path: '/key-influencers', label: 'Key Influencers' },
    ];

    return (
        <header className="bg-white dark:bg-slate-900 shadow-sm sticky top-0 z-10">
            <div className="container mx-auto px-4 sm:px-6 lg:px-8">
                <div className="flex items-center justify-between h-16">
                    <div className="flex items-center space-x-4">
                        <button
                            onClick={onToggleSidebar}
                            className="text-slate-600 dark:text-slate-400"
                        >
                            <span className="material-icons">menu</span>
                        </button>
                        <Link to="/" className="flex items-center gap-2">
                            <span className="material-icons text-primary text-3xl">business_center</span>
                            <span className="text-xl font-bold text-slate-900 dark:text-white">Corporate Data Explorer</span>
                        </Link>
                    </div>

                    <nav className="hidden lg:flex items-center space-x-1">
                        {/* Companies Menu */}
                        <div className="relative group">
                            <button className="flex items-center text-sm font-medium text-slate-600 hover:text-primary dark:text-slate-300 dark:hover:text-primary transition-colors px-3 py-2">
                                Companies <span className="material-icons text-base">expand_more</span>
                            </button>
                            <div className="absolute top-full left-1/2 -translate-x-1/2 mt-2 p-2 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 rounded-lg shadow-xl z-50 min-w-[200px] hidden group-hover:block">
                                {companyLinks.map(link => (
                                    <NavLink
                                        key={link.path}
                                        to={link.path}
                                        className={({isActive}) => `block px-4 py-2 rounded-md text-sm text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 ${isActive && 'bg-slate-100 dark:bg-slate-800 font-semibold'}`}
                                    >
                                        {link.label}
                                    </NavLink>
                                ))}
                            </div>
                        </div>

                        {/* Business Names Menu */}
                        <div className="relative group">
                            <button className="flex items-center text-sm font-medium text-slate-600 hover:text-primary dark:text-slate-300 dark:hover:text-primary transition-colors px-3 py-2">
                                Business Names <span className="material-icons text-base">expand_more</span>
                            </button>
                            <div className="absolute top-full left-1/2 -translate-x-1/2 mt-2 p-2 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 rounded-lg shadow-xl z-50 min-w-[200px] hidden group-hover:block">
                                {businessNameLinks.map(link => (
                                    <NavLink
                                        key={link.path}
                                        to={link.path}
                                        className={({isActive}) => `block px-4 py-2 rounded-md text-sm text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 ${isActive && 'bg-slate-100 dark:bg-slate-800 font-semibold'}`}
                                    >
                                        {link.label}
                                    </NavLink>
                                ))}
                            </div>
                        </div>

                        {/* Other Entities Menu */}
                        <div className="relative group">
                            <button className="flex items-center text-sm font-medium text-slate-600 hover:text-primary dark:text-slate-300 dark:hover:text-primary transition-colors px-3 py-2">
                                Other Entities <span className="material-icons text-base">expand_more</span>
                            </button>
                            <div className="absolute top-full left-1/2 -translate-x-1/2 mt-2 p-2 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 rounded-lg shadow-xl z-50 min-w-[200px] hidden group-hover:block">
                                {otherEntityLinks.map(link => (
                                    <NavLink
                                        key={link.path}
                                        to={link.path}
                                        className={({isActive}) => `block px-4 py-2 rounded-md text-sm text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 ${isActive && 'bg-slate-100 dark:bg-slate-800 font-semibold'}`}
                                    >
                                        {link.label}
                                    </NavLink>
                                ))}
                            </div>
                        </div>

                        {/* Contacts Menu */}
                        <div className="relative group">
                            <button className="flex items-center text-sm font-medium text-slate-600 hover:text-primary dark:text-slate-300 dark:hover:text-primary transition-colors px-3 py-2">
                                Contacts <span className="material-icons text-base">expand_more</span>
                            </button>
                            <div className="absolute top-full left-1/2 -translate-x-1/2 mt-2 p-2 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 rounded-lg shadow-xl z-50 min-w-[200px] hidden group-hover:block">
                                {contactLinks.map(link => (
                                    <NavLink
                                        key={link.path}
                                        to={link.path}
                                        className={({isActive}) => `block px-4 py-2 rounded-md text-sm text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 ${isActive && 'bg-slate-100 dark:bg-slate-800 font-semibold'}`}
                                    >
                                        {link.label}
                                    </NavLink>
                                ))}
                            </div>
                        </div>

                        {/* Visualizations Menu */}
                        <div className="relative group">
                            <button className="flex items-center text-sm font-medium text-slate-600 hover:text-primary dark:text-slate-300 dark:hover:text-primary transition-colors px-3 py-2">
                                Visualizations <span className="material-icons text-base">expand_more</span>
                            </button>
                            <div className="absolute top-full right-0 mt-2 p-2 bg-white dark:bg-slate-900 border border-slate-200 dark:border-slate-700 rounded-lg shadow-xl z-50 min-w-[200px] hidden group-hover:block">
                                {vizLinks.map(link => (
                                    <NavLink
                                        key={link.path}
                                        to={link.path}
                                        className={({isActive}) => `block px-4 py-2 rounded-md text-sm text-slate-700 dark:text-slate-300 hover:bg-slate-100 dark:hover:bg-slate-800 ${isActive && 'bg-slate-100 dark:bg-slate-800 font-semibold'}`}
                                    >
                                        {link.label}
                                    </NavLink>
                                ))}
                            </div>
                        </div>

                        {/* Approved Only Toggle */}
                        <ApprovedOnlyToggle className="pl-4" />
                    </nav>
                </div>
            </div>
        </header>
    );
};
```

### 9. Update Sidebar
Add the new link under sidebar sections.

**File:** `src/components/Sidebar.tsx`

```tsx
// leads_webapp1/src/components/Sidebar.tsx
import React from 'react';
import { NavLink, useLocation } from 'react-router-dom';
import { ApprovedOnlyToggle } from './ApprovedOnlyToggle';

const HomeIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path d="M10.707 2.293a1 1 0 00-1.414 0l-7 7a1 1 0 001.414 1.414L4 10.414V17a1 1 0 001 1h2a1 1 0 001-1v-2a1 1 0 011-1h2a1 1 0 011 1v2a1 1 0 001 1h2a1 1 0 001-1v-6.586l.293.293a1 1 0 001.414-1.414l-7-7z" /></svg>;
const UsersIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path d="M9 6a3 3 0 11-6 0 3 3 0 016 0zM17 6a3 3 0 11-6 0 3 3 0 016 0zM12.93 17c.046-.327.07-.66.07-1a6.97 6.97 0 00-1.5-4.33A5 5 0 0119 16v1h-6.07zM6 11a5 5 0 015 5v1H1v-1a5 5 0 015-5z" /></svg>;
const BriefcaseIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path fillRule="evenodd" d="M6 6V5a3 3 0 013-3h2a3 3 0 013 3v1h2a2 2 0 012 2v3.57A22.952 22.952 0 0110 13a22.95 22.95 0 01-8-1.43V8a2 2 0 012-2h2zm2-1a1 1 0 011-1h2a1 1 0 011 1v1H8V5zm1 5a1 1 0 011-1h.01a1 1 0 110 2H10a1 1 0 01-1-1z" clipRule="evenodd" /><path d="M2 13.692V16a2 2 0 002 2h12a2 2 0 002-2v-2.308A24.974 24.974 0 0110 15c-2.796 0-5.487-.46-8-1.308z" /></svg>;
const ContactsIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path fillRule="evenodd" d="M10 2a1 1 0 00-1 1v1a1 1 0 002 0V3a1 1 0 00-1-1zM4 4a1 1 0 00-1 1v12a1 1 0 001 1h12a1 1 0 001-1V5a1 1 0 00-1-1H4zM3 15a1 1 0 011-1h12a1 1 0 110 2H4a1 1 0 01-1-1zM8 6a1 1 0 000 2h4a1 1 0 100-2H8z" clipRule="evenodd" /></svg>;
const NetworkIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path d="M15 8a3 3 0 10-2.977-2.63l-4.94 2.47a3 3 0 100 4.319l4.94 2.47a3 3 0 10.895-1.789l-4.94-2.47a3.027 3.027 0 000-.74l4.94-2.47C13.456 7.68 14.19 8 15 8z" /></svg>;
const CorporateIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path fillRule="evenodd" d="M4 4a2 2 0 00-2 2v8a2 2 0 002 2h12a2 2 0 002-2V8a2 2 0 00-2-2h-5L9 4H4zm2 6a1 1 0 011-1h6a1 1 0 110 2H7a1 1 0 01-1-1z" clipRule="evenodd" /></svg>;
const InfluencerIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path fillRule="evenodd" d="M10 9a3 3 0 100-6 3 3 0 000 6zm-7 9a7 7 0 1114 0H3z" clipRule="evenodd" /></svg>;
const ReceiptIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path fillRule="evenodd" d="M5 4a3 3 0 00-3 3v6a3 3 0 003 3h10a3 3 0 003-3V7a3 3 0 00-3-3H5zm-1 9v-1h5v2H5a1 1 0 01-1-1zm7 1h4a1 1 0 001-1v-1h-5v2zm0-4h5V8h-5v2zM4 8h5v2H4V8zm0 5a1 1 0 011-1h5v2H4v-1z" clipRule="evenodd" /></svg>;
const GroupsIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path d="M13 6a3 3 0 11-6 0 3 3 0 016 0zM18 8a2 2 0 11-4 0 2 2 0 014 0zM14 15a4 4 0 00-8 0v3h8v-3zM6 8a2 2 0 11-4 0 2 2 0 014 0zM16 18v-3a5.972 5.972 0 00-.75-2.906A3.005 3.005 0 0119 15v3h-3zM4.75 12.094A5.973 5.973 0 004 15v3H1v-3a3 3 0 013.75-2.906z" /></svg>;
const PlaceIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path fillRule="evenodd" d="M5.05 4.05a7 7 0 119.9 9.9L10 20l-4.95-5.95a7 7 0 010-9.9zM10 11a2 2 0 100-4 2 2 0 000 4z" clipRule="evenodd" /></svg>;
const PeoplePlaceIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path fillRule="evenodd" d="M3 6a3 3 0 013-3h10a1 1 0 01.8 1.6L14.25 8l2.55 3.4A1 1 0 0116 13H6a1 1 0 00-1 1v3a1 1 0 11-2 0V6z" clipRule="evenodd" /></svg>;
const PeoplePlaceProvidedIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-5 w-5 flex-shrink-0" viewBox="0 0 20 20" fill="currentColor"><path fillRule="evenodd" d="M5.05 4.05a7 7 0 119.9 9.9L10 20l-4.95-5.95a7 7 0 010-9.9zM10 11a2 2 0 100-4 2 2 0 000 4z" clipRule="evenodd" /></svg>; // Reusing similar icon for now
const CloseIcon: React.FC = () => <svg xmlns="http://www.w3.org/2000/svg" className="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M6 18L18 6M6 6l12 12" /></svg>;

interface SidebarProps {
    isSidebarOpen: boolean;
    toggleSidebar: () => void;
}

const NavItem: React.FC<{ to: string; icon: React.ReactNode; label: string; onClick: () => void; end?: boolean; }> = ({ to, icon, label, onClick, end }) => {
    const activeClassName = "bg-primary text-white";
    const inactiveClassName = "hover:bg-slate-700 text-slate-300";

    return (
        <NavLink
            to={to}
            end={end}
            className={({ isActive }) =>
                `flex items-center gap-3 p-3 rounded-md font-semibold transition-colors duration-200 ${
                    isActive ? activeClassName : inactiveClassName
                }`
            }
            onClick={onClick}
        >
            {icon}
            <span className="whitespace-nowrap">{label}</span>
        </NavLink>
    );
};

export const Sidebar: React.FC<SidebarProps> = ({ isSidebarOpen, toggleSidebar }) => {
    const location = useLocation();

    return (
        <>
            <div
                className={`fixed inset-0 bg-black bg-opacity-60 z-30 transition-opacity duration-300 ease-in-out ${
                    isSidebarOpen ? 'opacity-100' : 'opacity-0 pointer-events-none'
                }`}
                onClick={toggleSidebar}
                aria-hidden="true"
            />
            <aside
                className={`fixed top-0 left-0 h-full w-64 bg-slate-800 border-r border-slate-700 transform transition-transform duration-300 ease-in-out z-40 ${
                    isSidebarOpen ? 'translate-x-0' : '-translate-x-full'
                }`}
            >
                <div className="flex flex-col h-full">
                    <div className="flex items-center justify-between p-4 border-b border-slate-700">
                        <h2 className="text-lg font-bold text-white">Menu</h2>
                        <button
                            onClick={toggleSidebar}
                            className="text-slate-400 hover:text-white"
                        >
                            <CloseIcon />
                        </button>
                    </div>

                    <nav className="flex-grow flex flex-col gap-2 p-2 overflow-y-auto">
                        <h3 className="px-3 pt-4 pb-2 text-xs font-bold text-slate-500 uppercase">Companies</h3>
                        <NavItem to={'/company/dashboard'} icon={<HomeIcon />} label="Dashboard" onClick={toggleSidebar} end />
                        <NavItem to={'/company/people'} icon={<UsersIcon />} label="People" onClick={toggleSidebar} />
                        <NavItem to={'/company/activities'} icon={<BriefcaseIcon />} label="Activities" onClick={toggleSidebar} />
                        <NavItem to={'/company/places'} icon={<PlaceIcon />} label="Place of Businesses" onClick={toggleSidebar} />
                        <NavItem to={'/company/people-places'} icon={<PeoplePlaceIcon />} label="Place of People (NIDA)" onClick={toggleSidebar} />
                        <NavItem to={'/company/people-places-provided'} icon={<PeoplePlaceProvidedIcon />} label="Place of People (As Provided)" onClick={toggleSidebar} />

                        <h3 className="px-3 pt-4 pb-2 text-xs font-bold text-slate-500 uppercase">Business Names</h3>
                        <NavItem to={'/business-name/dashboard'} icon={<HomeIcon />} label="Dashboard" onClick={toggleSidebar} end />
                        <NavItem to={'/business-name/people'} icon={<UsersIcon />} label="People" onClick={toggleSidebar} />
                        <NavItem to={'/business-name/activities'} icon={<BriefcaseIcon />} label="Activities" onClick={toggleSidebar} />
                        <NavItem to={'/business-name/places'} icon={<PlaceIcon />} label="Place of Businesses" onClick={toggleSidebar} />
                        <NavItem to={'/business-name/people-places'} icon={<PeoplePlaceIcon />} label="Place of People (NIDA)" onClick={toggleSidebar} />
                        <NavItem to={'/business-name/people-places-provided'} icon={<PeoplePlaceProvidedIcon />} label="Place of People (As Provided)" onClick={toggleSidebar} />

                        <h3 className="px-3 pt-4 pb-2 text-xs font-bold text-slate-500 uppercase">Other Entities</h3>
                        <NavItem to="/corporate-shareholders" icon={<CorporateIcon />} label="Corporate Shareholders" onClick={toggleSidebar} />
                        <NavItem to="/tra-tins" icon={<ReceiptIcon />} label="TRA Tins" onClick={toggleSidebar} />
                        <NavItem to="/wabunge" icon={<GroupsIcon />} label="Wabunge" onClick={toggleSidebar} />

                        <h3 className="px-3 pt-4 pb-2 text-xs font-bold text-slate-500 uppercase">Shared</h3>
                        <NavItem to="/contacts" icon={<ContactsIcon />} label="Contact Directory" onClick={toggleSidebar} />
                        <NavItem to="/phone-filter" icon={<ContactsIcon />} label="Phone Filter" onClick={toggleSidebar} />

                        <h3 className="px-3 pt-4 pb-2 text-xs font-bold text-slate-500 uppercase">Visualizations</h3>
                        <NavItem to="/network-graph" icon={<NetworkIcon />} label="Network Graph" onClick={toggleSidebar} />
                        <NavItem to="/key-influencers" icon={<InfluencerIcon />} label="Key Influencers" onClick={toggleSidebar} />

                        {/* Global Filter Section */}
                        <div className="mt-auto p-2">
                            <ApprovedOnlyToggle className="bg-slate-700/50 p-3 rounded-lg" />
                        </div>
                    </nav>
                </div>
            </aside>
        </>
    );
};
```

### 10. Update App Routes
Register the new page component.

**File:** `src/App.tsx`

```tsx
// leads_webapp1/src/App.tsx
import React, { useState } from 'react';
import { HashRouter, Routes, Route, Navigate } from 'react-router-dom';
import { Header } from './components/Header';
import { Sidebar } from './components/Sidebar';
import PeoplePage from './pages/PeoplePage';
import PersonDetailPage from './pages/PersonDetailPage';
import BusinessActivitiesPage from './pages/BusinessActivitiesPage';
import BusinessActivityRegistrationsPage from './pages/BusinessActivityRegistrationsPage';
import ContactDirectoryPage from './pages/ContactDirectoryPage';
import ListPage from './pages/ListPage';
import CompanyDetailPage from './pages/details/CompanyDetailPage';
import BusinessNameDetailPage from './pages/details/BusinessNameDetailPage';
import NetworkGraphPage from './pages/NetworkGraphPage/index';
import CorporateShareholdersPage from './pages/CorporateShareholdersPage';
import CorporateShareholderDetailPage from './pages/details/CorporateShareholderDetailPage';
import KeyInfluencersPage from './pages/KeyInfluencersPage';
import PhoneFilterPage from './pages/PhoneFilterPage';
import TraTinsPage from './pages/TraTinsPage';
import TRAtinsDetailPage from './pages/details/TRAtinsDetailPage';
import WabungePage from './pages/WabungePage';
import MbungeDetailPage from './pages/MbungeDetailPage';
import PlacesPage from './pages/PlacesPage';
import RegistrationsByPlacePage from './pages/RegistrationsByPlacePage';
import PlaceOfPeoplePage from './pages/PlaceOfPeoplePage';
import PlaceOfPeopleAsProvidedPage from './pages/PlaceOfPeopleAsProvidedPage'; // Import new page

const App: React.FC = () => {
    const [isSidebarOpen, setIsSidebarOpen] = useState(false);

    const toggleSidebar = () => setIsSidebarOpen(prev => !prev);

    return (
        <HashRouter>
            <div className="flex flex-col min-h-screen">
                <Header onToggleSidebar={toggleSidebar} />
                <Sidebar isSidebarOpen={isSidebarOpen} toggleSidebar={toggleSidebar} />
                <main className="flex-grow container mx-auto px-4 sm:px-6 lg:px-8 py-8">
                    <Routes>
                        <Route path="/" element={<Navigate to="/company/dashboard" replace />} />

                        {/* Company Routes */}
                        <Route path="/company/dashboard" element={<ListPage registrationType="company" />} />
                        <Route path="/company/people" element={<PeoplePage registrationType="company" />} />
                        <Route path="/company/activities" element={<BusinessActivitiesPage registrationType="company" />} />
                        <Route path="/company/activity/:activity" element={<BusinessActivityRegistrationsPage registrationType="company" />} />
                        <Route path="/company/places" element={<PlacesPage registrationType="company" />} />
                        <Route path="/company/place/:placeId" element={<RegistrationsByPlacePage registrationType="company" />} />
                        <Route path="/company/people-places" element={<PlaceOfPeoplePage registrationType="company" />} />
                        <Route path="/company/people-places-provided" element={<PlaceOfPeopleAsProvidedPage registrationType="company" />} /> {/* New Route */}
                        <Route path="/company/detail/:id" element={<CompanyDetailPage />} />

                        {/* Business Name Routes */}
                        <Route path="/business-name/dashboard" element={<ListPage registrationType="business-name" />} />
                        <Route path="/business-name/people" element={<PeoplePage registrationType="business-name" />} />
                        <Route path="/business-name/activities" element={<BusinessActivitiesPage registrationType="business-name" />} />
                        <Route path="/business-name/activity/:activity" element={<BusinessActivityRegistrationsPage registrationType="business-name" />} />
                        <Route path="/business-name/places" element={<PlacesPage registrationType="business-name" />} />
                        <Route path="/business-name/place/:placeId" element={<RegistrationsByPlacePage registrationType="business-name" />} />
                        <Route path="/business-name/people-places" element={<PlaceOfPeoplePage registrationType="business-name" />} />
                        <Route path="/business-name/people-places-provided" element={<PlaceOfPeopleAsProvidedPage registrationType="business-name" />} /> {/* New Route */}
                        <Route path="/business-name/detail/:id" element={<BusinessNameDetailPage />} />

                        {/* Corporate Shareholder Routes */}
                        <Route path="/corporate-shareholders" element={<CorporateShareholdersPage />} />
                        <Route path="/corporate-shareholder/:id" element={<CorporateShareholderDetailPage />} />

                        {/* TRA Tins Routes */}
                        <Route path="/tra-tins" element={<TraTinsPage />} />
                        <Route path="/tra-tin/:id" element={<TRAtinsDetailPage />} />

                        {/* Wabunge Routes */}
                        <Route path="/wabunge" element={<WabungePage />} />
                        <Route path="/wabunge/:id" element={<MbungeDetailPage />} />

                        {/* Shared & Tool Routes */}
                        <Route path="/contacts" element={<ContactDirectoryPage />} />
                        <Route path="/phone-filter" element={<PhoneFilterPage />} />
                        <Route path="/person/:personId" element={<PersonDetailPage />} />
                        <Route path="/network-graph" element={<NetworkGraphPage />} />
                        <Route path="/key-influencers" element={<KeyInfluencersPage />} />

                        <Route path="*" element={<Navigate to="/company/dashboard" replace />} />
                    </Routes>
                </main>
            </div>
        </HashRouter>
    );
};

export default App;
```

### 11. Update ApprovedOnlyToggle
Ensure the toggle is visible on the new routes.

**File:** `src/components/ApprovedOnlyToggle.tsx`

```tsx
// leads_webapp1/src/components/ApprovedOnlyToggle.tsx
import React from 'react';
import { useLocation, useSearchParams } from 'react-router-dom';
import { ToggleSwitch } from './ToggleSwitch';

// List of paths where the "Approved Only" toggle should be visible
const visiblePaths = [
    '/company/dashboard',
    '/business-name/dashboard',
    '/person/',
    '/key-influencers',
    '/contacts',
    '/corporate-shareholder/',
    '/company/places',
    '/business-name/places',
    '/company/place/',
    '/business-name/place/',
    '/company/people-places',
    '/business-name/people-places',
    '/company/people-places-provided', // NEW
    '/business-name/people-places-provided', // NEW
];

export const ApprovedOnlyToggle: React.FC<{ className?: string }> = ({ className }) => {
    const location = useLocation();
    const [searchParams, setSearchParams] = useSearchParams();

    const isVisible = visiblePaths.some(path => location.pathname.startsWith(path));

    if (!isVisible) {
        return null;
    }

    const approvedOnly = searchParams.get('approvedOnly') === 'true';

    const handleToggle = (enabled: boolean) => {
        setSearchParams(prev => {
            if (enabled) {
                prev.set('approvedOnly', 'true');
            } else {
                prev.delete('approvedOnly');
            }
            // Reset page to 1 when filter changes, if page exists
            if (prev.has('page')) {
                prev.set('page', '1');
            }
            return prev;
        }, { replace: true });
    };

    return (
        <div className={className}>
            <ToggleSwitch label="Approved Only" enabled={approvedOnly} onChange={handleToggle} />
        </div>
    );
};
```
