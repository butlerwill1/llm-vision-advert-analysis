# Ad Creative Vision Analysis

AI-powered visual analysis tool for social media advertisements using GPT-4 Vision. Automatically extracts creative insights including emotion detection, color analysis, object recognition, and engagement prediction from ad imagery.

## Overview

This project implements an end-to-end data pipeline that leverages OpenAI's GPT-4 Vision API to analyze social media advertisement creatives at scale. The system processes image assets, extracts structured metadata, and stores results in a NoSQL database for downstream analytics and reporting.

### Key Features

- **Multimodal LLM Integration**: Utilizes GPT-4 Vision (gpt-4o) for image understanding and structured data extraction
- **Automated Creative Analysis**: Extracts 11+ dimensions of creative metadata including emotion, color, objects, demographics, and engagement potential
- **Scalable Data Pipeline**: Batch processing with retry logic, exponential backoff, and rate limit handling
- **Multi-Database Architecture**: Integrates Snowflake (data warehouse), AWS DocumentDB (NoSQL storage), and social media APIs
- **Emotion Intelligence**: Implements Plutchik's Wheel of Emotions for standardized emotional classification
- **Production-Ready Error Handling**: Comprehensive HTTP error handling with configurable retry strategies

## Architecture

### Data Flow

1. **Data Ingestion**: Social media post data extracted from Snowflake data warehouse (RivalIQ Instagram dataset)
2. **Image Acquisition**: Automated download and preprocessing of advertisement images
3. **Vision Analysis**: Base64-encoded images sent to GPT-4 Vision API with structured prompts
4. **Data Storage**: JSON responses stored in AWS DocumentDB for flexible schema management
5. **Analytics Layer**: Aggregated insights written back to Snowflake for BI tool consumption

### Technology Stack

- **Language**: Python 3.x
- **LLM Provider**: OpenAI GPT-4 Vision (gpt-4o)
- **Data Warehouse**: Snowflake
- **NoSQL Database**: AWS DocumentDB (MongoDB-compatible)
- **Image Processing**: Pillow (PIL)
- **API Client**: requests with custom retry logic
- **Data Manipulation**: pandas

## Installation

### Prerequisites

- Python 3.8+
- OpenAI API key with GPT-4 Vision access
- Snowflake account and credentials
- AWS DocumentDB cluster (or MongoDB instance)
- SSL certificate for DocumentDB connection (`global-bundle.pem`)

### Setup

1. Clone the repository:
```bash
git clone https://github.com/yourusername/gpt_vision_ad_analysis.git
cd gpt_vision_ad_analysis
```

2. Install dependencies:
```bash
pip install openai pandas pymongo pillow python-dotenv requests snowflake-connector-python
```

3. Configure environment variables in `.env`:
```bash
# OpenAI
openapi_key=your_openai_api_key

# Snowflake
snowflake_account_soc=your_account
snowflake_warehouse_soc=your_warehouse
snowflake_database_soc=your_database
snowflake_schema_soc=your_schema
snowflake_user_soc=your_user
snowflake_password_soc=your_password
snowflake_user_role_soc=your_role
```

4. Place AWS DocumentDB SSL certificate in project root:
```bash
wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
```

## Usage

### 1. Download Advertisement Images

Extract social media posts from Snowflake and download associated images:

```python
python file_downloads.py
```

This script:
- Queries Instagram post data from Snowflake
- Filters and samples posts by company/brand
- Downloads images with standardized naming convention
- Implements rate limiting to respect API constraints

### 2. Run Vision Analysis

Process images through GPT-4 Vision API:

```python
python openai_analysis.py
```

Key operations:
- Loads images from local directory
- Encodes images as base64 with size optimization (512x512)
- Sends structured prompts to GPT-4 Vision API
- Stores JSON responses in DocumentDB with metadata

### 3. Aggregate and Export Results

Combine vision analysis results with performance metrics:

```python
python nosql_analysis.py
```

This pipeline:
- Retrieves analysis results from DocumentDB
- Joins with original post performance data (impressions, engagement)
- Calculates aggregate statistics by creative dimensions
- Writes final dataset to Snowflake for BI consumption

## Data Schema

### Vision Analysis Output

Each image analysis returns structured JSON with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `Category` | String | Post classification (Product, Lifestyle, Event, Infographic, Meme, Personal, Promotional) |
| `DominantColour` | String | Primary color from predefined palette (Red, Orange, Yellow, Green, Blue, Purple, Pink, Brown, Gray, Black, White) |
| `Objects` | Array | List of prominent objects detected (person, animal, vehicle, product, etc.) |
| `TextPresence` | Boolean | Whether text overlay is present in image |
| `NumberOfPeople` | Integer | Count of people visible in image |
| `GenderOfPeople` | Array | Gender classification for each person detected |
| `EstimatedAges` | Array | Age estimates for people in image |
| `Emotion` | String | Dominant emotion based on Plutchik's Wheel |
| `EmotionalIntensity` | Integer | Emotion strength rating (1-10 scale) |
| `Context` | String | Brief scene description |
| `EngagementPotential` | Integer | Predicted social media engagement score (1-10) |

### Sample Response

```json
{
  "id": "chatcmpl-9Zi8oNpNCp9U5gYs3oZ2BkJyJSNpc",
  "model": "gpt-4o-2024-05-13",
  "choices": [{
    "message": {
      "content": {
        "Category": "Lifestyle",
        "DominantColour": "Green",
        "Objects": ["Person", "Sunglasses", "Blazer", "Pants", "Bag"],
        "TextPresence": false,
        "NumberOfPeople": 1,
        "GenderOfPeople": ["Female"],
        "EstimatedAges": [28],
        "Emotion": "Calm",
        "EmotionalIntensity": 6,
        "Context": "Well-dressed person standing in front of building with green doors",
        "EngagementPotential": 8
      }
    }
  }],
  "usage": {
    "prompt_tokens": 575,
    "completion_tokens": 154,
    "total_tokens": 729
  },
  "image_name": "anthropologie-ig-p-C4fm7TvMCuv.jpg",
  "response_date": "2024-06-13T14:23:06"
}
```

## Core Components

### CreativeAnalytics Class

Central class handling API interactions with production-grade error handling:

**Key Methods:**

- `image_prompt_openai()`: Sends image + prompt to GPT-4 Vision API with structured JSON response format
- `run_request_with_error_handling()`: Wrapper for HTTP requests with retry logic
- `exponential_backoff_delay()`: Implements exponential backoff with jitter for rate limit handling

**Error Handling Strategy:**

- **Rate Limits (429)**: Exponential backoff with jitter
- **Permanent Errors (400, 401, 403)**: Configurable stop or retry behavior
- **Temporary Errors (500, 502, 503, 504)**: Automatic retry with backoff
- **Max Retries**: Configurable threshold (default: 4 attempts)

### Utility Functions

- `resize_encode_image()`: Optimizes image size (512x512) and converts to base64 encoding
- `download_image_requests()`: Downloads images with MIME type detection
- `url_shortener()`: Generates standardized image filenames from social media URLs
- `company_name_cleaner()`: Normalizes brand names for consistent grouping

## Performance Considerations

### API Cost Optimization

- **Image Resizing**: All images resized to 512x512 before encoding to reduce token consumption
- **Batch Processing**: Processes images in configurable batches to manage API costs
- **Response Caching**: Stores all API responses in DocumentDB to prevent duplicate analysis

### Rate Limit Management

- Implements exponential backoff: `delay = (4^retry_count + 1) * 5 seconds`
- Adds random jitter (0-10% of delay) to prevent thundering herd
- Monitors rate limit headers: `RateLimit-Limit`, `RateLimit-Remaining`, `RateLimit-Reset`

### Data Pipeline Efficiency

- **Incremental Processing**: Tracks processed images to avoid reanalysis
- **Direct Connection Mode**: Uses DocumentDB direct connection for improved latency
- **Snowflake Bulk Operations**: Leverages `write_df_to_snowflake()` for efficient data loading

## Analytics Use Cases

### Creative Performance Analysis

Correlate creative attributes with engagement metrics:

```python
# Aggregate by emotion
emotion_stats = df.groupby('Emotion').agg({
    'estimated_impressions': ['median', 'mean'],
    'applause': ['median', 'mean'],
    'post_id': 'count'
})
```

### A/B Testing Insights

Compare creative variations:
- Color palette impact on engagement
- Emotional tone effectiveness by audience segment
- Optimal number of people in frame
- Text overlay vs. image-only performance

### Brand Consistency Monitoring

Track creative attributes across campaigns:
- Dominant color distribution over time
- Emotion profile consistency
- Category mix analysis

### Competitive Intelligence

Benchmark creative strategies:
- Compare emotion usage across competitors
- Identify trending visual patterns
- Analyze category distribution by brand

## Project Structure

```
.
├── functions.py              # Core utilities and CreativeAnalytics class
├── prompts.py               # GPT-4 Vision prompt templates
├── file_downloads.py        # Image acquisition from Snowflake/APIs
├── openai_analysis.py       # Vision API processing pipeline
├── nosql_analysis.py        # DocumentDB to Snowflake ETL
├── sample_openai_response.json  # Example API response
├── images/                  # Downloaded advertisement images
└── global-bundle.pem        # AWS DocumentDB SSL certificate
```

## Configuration

### Snowflake Connection

The project uses two Snowflake connection profiles:

1. **RivalIQ Read-Only**: For accessing social media data
2. **SOC Write Access**: For writing analysis results

### DocumentDB Connection

Requires SSL/TLS connection with certificate validation:

```python
cluster_client = MongoClient(
    host=cluster_endpoint,
    tls=True,
    tlsCAFile='global-bundle.pem',
    tlsAllowInvalidHostnames=True,
    username=username,
    password=password,
    directConnection=True,
    authSource='admin',
    retryWrites=False
)
```

## Limitations and Future Enhancements

### Current Limitations

- Single-image analysis only (no video or carousel support)
- English-language prompts and responses
- Fixed emotion taxonomy (Plutchik's Wheel)
- Manual batch size configuration

### Potential Enhancements

- **Multi-modal Analysis**: Extend to video content with frame sampling
- **Real-time Processing**: Implement streaming pipeline with message queue
- **Advanced Analytics**: Add clustering, trend detection, and anomaly identification
- **Model Comparison**: A/B test different vision models (GPT-4V vs. Claude 3 vs. Gemini)
- **Automated Reporting**: Generate executive dashboards with key insights
- **Cost Tracking**: Implement per-image cost monitoring and budget alerts

## License

This project is provided as-is for educational and commercial use.

## Acknowledgments

- OpenAI for GPT-4 Vision API
- RivalIQ for social media intelligence data
- Plutchik's Wheel of Emotions for emotion taxonomy


