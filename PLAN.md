# Burry AI Implementation Plan

## Phase 1: Project Setup and Authentication

### 1.1 Project Initialization
- [ ] Initialize Rails application with PostgreSQL
- [ ] Set up Git repository
- [ ] Configure Ruby version and gemfile with required dependencies
- [ ] Set up development environment with required gems
- [ ] Configure CI/CD pipeline with GitHub Actions

### 1.2 Testing Framework Setup
- [ ] Configure RSpec with Rails
- [ ] Set up factory_bot_rails
- [ ] Configure VCR and WebMock for API testing
- [ ] Set up Faker for test data generation
- [ ] Configure SimpleCov for test coverage reporting

### 1.3 Google OAuth Integration
- [ ] Set up OmniAuth gem with Google provider
- [ ] Create Google Cloud project for OAuth
- [ ] Configure OAuth callback URLs
- [ ] Implement sign in with Google functionality
- [ ] Test authentication flow successfully logs in user

### 1.4 User Model Implementation
- [ ] Create User model with required attributes
- [ ] Implement user authentication methods
- [ ] Set up routes for authentication
- [ ] Create login/logout functionality
- [ ] Write unit tests that verify authentication flow

## Phase 2: Minimal Google Ads Integration

### 2.1 Google Ads API Setup
- [ ] Register application with Google Ads API
- [ ] Set up API credentials
- [ ] Configure API client
- [ ] Test API connectivity
- [ ] Document API integration

### 2.2 Core Data Models
- [ ] Create GoogleAdAccount model
- [ ] Create GoogleCampaign model
- [ ] Create minimal GoogleDailyMetric model (just spend)
- [ ] Set up polymorphic relationship for metrics
- [ ] Write tests for model relationships

### 2.3 OAuth Connection Flow
- [ ] Implement OAuth authorization flow for Google Ads
- [ ] Create connection management interface
- [ ] Store and refresh OAuth tokens
- [ ] Handle connection errors
- [ ] Test OAuth connection flow end-to-end

### 2.4 Minimal Data Fetching
- [ ] Implement service to fetch account data
- [ ] Create campaign data fetching methods
- [ ] Implement daily spend fetching
- [ ] Create data storage logic
- [ ] Test data fetching with VCR cassettes

## Phase 3: Minimal Data Refresh System

### 3.1 Background Job Setup
- [ ] Set up GoodJob for background processing
- [ ] Configure job queues
- [ ] Create job failure logging
- [ ] Set up job monitoring
- [ ] Test job framework operation

### 3.2 Basic Refresh Implementation
- [ ] Create refresh job structure
- [ ] Implement daily spend data fetch
- [ ] Create refresh initiation endpoint
- [ ] Log refresh completion status
- [ ] Test refresh process fetches and stores data correctly

## Phase 4: LLM Integration

### 4.1 LLM Provider Selection
- [ ] Research available LLM providers
- [ ] Compare pricing and feature models
- [ ] Evaluate response quality
- [ ] Select initial LLM provider
- [ ] Document selection criteria

### 4.2 Basic LLM Integration
- [ ] Create LLM provider client
- [ ] Set up API authentication
- [ ] Implement simple prompt template
- [ ] Configure request parameters
- [ ] Test LLM providing responses about spend

### 4.3 Minimal Prompt Engineering
- [ ] Create basic system prompt template
- [ ] Implement spend data injection
- [ ] Create prompt rendering logic
- [ ] Test prompt includes accurate spend data
- [ ] Verify LLM responses contain spend information

## Phase 5: Minimal Web Interface

### 5.1 Frontend Basics
- [ ] Configure frontend dependencies
- [ ] Set up CSS framework
- [ ] Create base layout component
- [ ] Implement responsive design
- [ ] Create simple nav menu

### 5.2 Minimal Dashboard View
- [ ] Create dashboard view
- [ ] Implement account selection
- [ ] Display campaign spend data
- [ ] Show data freshness indicators
- [ ] Create refresh button

### 5.3 Basic Chat Interface
- [ ] Create simple chat UI
- [ ] Implement message input
- [ ] Display message history
- [ ] Connect to LLM backend
- [ ] Test queries about spend data

## Phase 6: Deployment

### 6.1 Heroku Setup
- [ ] Create Heroku application
- [ ] Configure production environment variables 
- [ ] Set up database on Heroku
- [ ] Configure application logging
- [ ] Set up DNS for custom domain

### 6.2 Initial Deployment
- [ ] Configure production environment
- [ ] Run database migrations
- [ ] Deploy application to Heroku
- [ ] Verify authentication works in production
- [ ] Test Google OAuth flow in production

### 6.3 Production Validation
- [ ] Verify Google Ads API connectivity
- [ ] Test data fetching in production
- [ ] Validate LLM integration
- [ ] Confirm end-to-end functionality
- [ ] Document deployment process

## Phase 7: Slack Integration

### 7.1 Slack API Setup
- [ ] Register Slack application
- [ ] Configure bot permissions
- [ ] Set up OAuth for Slack
- [ ] Test Slack API connectivity
- [ ] Document Slack integration process

### 7.2 Message Relay Implementation
- [ ] Create SlackIntegration model
- [ ] Implement Slack event handling
- [ ] Create message relay service to LLM
- [ ] Handle conversation threading
- [ ] Test message relay functionality

### 7.3 Response Formatting
- [ ] Implement response formatting for Slack
- [ ] Create spend data visualization templates
- [ ] Handle message length limitations
- [ ] Format error responses
- [ ] Test end-to-end Slack interaction

## Phase 8: Extending Initial Implementation

### 8.1 Extended Data Models
- [ ] Create GoogleAdGroup model
- [ ] Create GoogleAd model
- [ ] Extend GoogleDailyMetric with additional metrics
- [ ] Update model relationships
- [ ] Add model annotations

### 8.2 Enhanced Data Fetching
- [ ] Implement ad group data retrieval
- [ ] Create ad performance data fetching
- [ ] Extend metrics collection beyond spend
- [ ] Implement efficient batch processing
- [ ] Test extended data model population

### 8.3 Enhanced Background Jobs
- [ ] Implement 6-hour refresh cycle
- [ ] Create scheduled refresh job
- [ ] Configure job concurrency settings
- [ ] Add job status tracking
- [ ] Test automated refresh cycle

### 8.4 LLM Prompt Enhancement
- [ ] Extend prompt templates for additional metrics
- [ ] Create hierarchical data presentation
- [ ] Implement performance comparison prompts
- [ ] Add time period handling
- [ ] Test enhanced prompt responses

## Phase 9: Testing and Documentation

### 9.1 Test Coverage Completion
- [ ] Ensure 80% unit test coverage
- [ ] Implement key integration tests
- [ ] Create end-to-end test suite
- [ ] Test cross-component functionality
- [ ] Fix any identified test failures

### 9.2 Performance Optimization
- [ ] Analyze query performance
- [ ] Optimize database indexes
- [ ] Implement query caching
- [ ] Measure and improve response times
- [ ] Document performance metrics

### 9.3 Documentation Completion
- [ ] Create user documentation
- [ ] Document API endpoints
- [ ] Create technical architecture documentation
- [ ] Document operational procedures
- [ ] Complete development handover documentation
