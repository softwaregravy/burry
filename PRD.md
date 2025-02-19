# Burry AI Product Requirements Document

## 1. Product Overview
Burry AI is a chatbot service that provides ad performance insights through natural language interaction, initially focusing on Google Ads with plans to expand to Facebook, TikTok, and other platforms.

## 2. System Architecture
- Backend database refreshed every 6 hours from ad platform APIs
- Facts-based prompt generation for LLM interaction
- Limited tool capabilities: refresh data and extend lookback period
- Standalone web interface with Slack integration

## 3. Core Features

### 3.1 Account Management
- User registration and authentication
- Google Ads account connection with OAuth
- Optional Slack workspace/channel integration

### 3.2 Data Collection
- 6-hour refresh cycle from Google Ads API
- 30-day default data lookback
- Optional 90-day extended lookback
- Transparent data freshness indicators

### 3.3 Performance Reporting
- Account-level metrics (spend, impressions, CTR, etc.)
- Hierarchical focus on highest performers:
  - Top 5 spending campaigns
  - Top 5 spending ad groups per campaign
  - Top 5 spending ads per ad group
- Daily summary template with key metrics
- Data timestamp visibility

### 3.4 Interaction Model
- Natural language queries about ad performance
- Pre-calculated metrics to ensure accuracy
- Refresh capability for latest data
- Extended lookback requests

## 4. Interface Requirements

### 4.1 Web Interface
- Setup and configuration dashboard
- Chat interface for direct interaction
- Connection management for ad platforms and Slack
- Authentication and security controls

### 4.2 Slack Integration
- Channel-specific bot presence
- Daily performance summaries
- Query responses in thread
- Data freshness indicators

## 5. Technical Requirements
- Google Ads API integration
- Database for storing periodic snapshots
- LLM integration with facts-based prompting
- Slack API integration
- Authentication and security infrastructure

## 6. Future Roadmap
- Custom daily summary templates
- User preference learning
- Campaign categorization and tagging
- Budget modification capabilities
- Deep links to ad platform
- Experiment and creative performance reporting
- Targeting configuration reports
- Additional ad platform integrations
