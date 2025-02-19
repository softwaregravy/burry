# Burry AI Technical Design Document

## 1. Database Schema Design

### 1.1 Core Entity Models

#### Users and Authentication
```ruby
class User < ApplicationRecord
  has_many :user_ad_accounts
  has_many :google_ad_accounts, through: :user_ad_accounts, source: :ad_account, source_type: 'GoogleAdAccount'
  has_many :facebook_ad_accounts, through: :user_ad_accounts, source: :ad_account, source_type: 'FacebookAdAccount'
  has_many :slack_integrations
  
  # Authentication fields
  t.string :email, null: false, index: { unique: true }
  t.string :encrypted_password, null: false
  t.string :reset_password_token
  t.datetime :reset_password_sent_at
  t.datetime :remember_created_at
  t.timestamps
end
```

#### Ad Account Polymorphic Association
```ruby
class UserAdAccount < ApplicationRecord
  belongs_to :user
  belongs_to :ad_account, polymorphic: true
  
  t.references :user, null: false, foreign_key: true
  t.references :ad_account, polymorphic: true, null: false
  t.timestamps
end
```

#### Google Ads Integration
```ruby
class GoogleAdAccount < ApplicationRecord
  has_many :user_ad_accounts, as: :ad_account
  has_many :users, through: :user_ad_accounts
  has_many :google_campaigns
  has_many :refresh_jobs, as: :refreshable
  
  t.string :account_id, null: false
  t.string :name
  t.string :oauth_refresh_token
  t.datetime :token_expires_at
  t.string :customer_id
  t.datetime :last_refreshed_at
  t.jsonb :account_settings
  t.timestamps
end

class GoogleCampaign < ApplicationRecord
  belongs_to :google_ad_account
  has_many :google_ad_groups
  has_many :google_daily_metrics, as: :measurable
  
  t.references :google_ad_account, null: false, foreign_key: true
  t.string :campaign_id, null: false
  t.string :name
  t.string :status
  t.string :campaign_type
  t.datetime :start_date
  t.datetime :end_date
  t.timestamps
end

class GoogleAdGroup < ApplicationRecord
  belongs_to :google_campaign
  has_many :google_ads
  has_many :google_daily_metrics, as: :measurable
  
  t.references :google_campaign, null: false, foreign_key: true
  t.string :ad_group_id, null: false
  t.string :name
  t.string :status
  t.timestamps
end

class GoogleAd < ApplicationRecord
  belongs_to :google_ad_group
  has_many :google_daily_metrics, as: :measurable
  
  t.references :google_ad_group, null: false, foreign_key: true
  t.string :ad_id, null: false
  t.string :headline
  t.text :description
  t.string :status
  t.jsonb :ad_details
  t.timestamps
end
```

#### Facebook/Meta Ads Integration (Future)
```ruby
class FacebookAdAccount < ApplicationRecord
  has_many :user_ad_accounts, as: :ad_account
  has_many :users, through: :user_ad_accounts
  has_many :facebook_campaigns
  has_many :refresh_jobs, as: :refreshable
  
  t.string :account_id, null: false
  t.string :name
  t.string :oauth_access_token
  t.datetime :token_expires_at
  t.datetime :last_refreshed_at
  t.jsonb :account_settings
  t.timestamps
end

# Similar structures for FacebookCampaign, FacebookAdSet, FacebookAd
```

### 1.2 Metrics and Performance Data

#### Time-Series Performance Data
```ruby
class GoogleDailyMetric < ApplicationRecord
  belongs_to :measurable, polymorphic: true
  
  t.references :measurable, polymorphic: true, null: false
  t.date :date, null: false
  t.decimal :impressions, precision: 12, scale: 2, default: 0
  t.decimal :clicks, precision: 12, scale: 2, default: 0
  t.decimal :cost, precision: 12, scale: 2, default: 0
  t.decimal :conversions, precision: 12, scale: 2, default: 0
  t.decimal :conversion_value, precision: 12, scale: 2, default: 0
  t.decimal :ctr, precision: 8, scale: 4
  t.decimal :cpc, precision: 8, scale: 2
  t.decimal :cpa, precision: 8, scale: 2
  t.decimal :roas, precision: 8, scale: 2
  t.jsonb :additional_metrics
  t.timestamps
  
  # Composite index for efficient querying
  t.index [:measurable_type, :measurable_id, :date], unique: true, name: 'idx_google_daily_metrics_composite'
end
```

### 1.3 Slack Integration Models
```ruby
class SlackIntegration < ApplicationRecord
  belongs_to :user
  
  t.references :user, null: false, foreign_key: true
  t.string :workspace_id
  t.string :channel_id
  t.string :bot_token
  t.string :installation_state
  t.boolean :daily_summary_enabled, default: true
  t.string :summary_schedule, default: '09:00'
  t.timestamps
end
```

### 1.4 Job and Processing Models
```ruby
class RefreshJob < ApplicationRecord
  belongs_to :refreshable, polymorphic: true
  belongs_to :user
  
  t.references :refreshable, polymorphic: true, null: false
  t.references :user, null: false, foreign_key: true
  t.string :status, default: 'pending'
  t.datetime :started_at
  t.datetime :completed_at
  t.text :error_message
  t.integer :days_lookback, default: 30
  t.timestamps
end

class LlmPrompt < ApplicationRecord
  belongs_to :user
  
  t.references :user, null: false, foreign_key: true
  t.text :prompt_template
  t.text :rendered_prompt
  t.jsonb :context_data
  t.datetime :generated_at
  t.datetime :expires_at
  t.boolean :is_active, default: true
  t.timestamps
  
  # Index for quick retrieval of active prompts
  t.index [:user_id, :is_active, :expires_at], name: 'idx_active_user_prompts'
end
```

### 1.5 Feedback and Monitoring
```ruby
class UserInteraction < ApplicationRecord
  belongs_to :user
  
  t.references :user, null: false, foreign_key: true
  t.text :user_query
  t.text :system_response
  t.string :interaction_source # 'web' or 'slack'
  t.boolean :marked_as_hallucination, default: false
  t.integer :user_satisfaction_rating
  t.text :user_feedback
  t.jsonb :context_data # Stores the specific data used to generate the response
  t.timestamps
end
```

## 2. Data Refresh Architecture

### 2.1 Regular Refresh Cycle

The system will implement a 6-hour refresh cycle using GoodJob background jobs with the following components:

```ruby
class ScheduledRefreshJob < ApplicationJob
  extend GoodJob::ActiveJobExtensions::Concurrency
  good_job_control_concurrency_with(
    key: -> { 'scheduled_refresh' },
    total_limit: 1
  )
  
  def perform
    # Group users with accounts needing refresh
    User.joins(:google_ad_accounts)
      .where('google_ad_accounts.last_refreshed_at < ?', 6.hours.ago)
      .distinct.find_each do |user|
      UserAccountsRefreshJob.perform_later(user.id, 'google')
    end
    
    # Future platforms
    User.joins(:facebook_ad_accounts)
      .where('facebook_ad_accounts.last_refreshed_at < ?', 6.hours.ago)
      .distinct.find_each do |user|
      UserAccountsRefreshJob.perform_later(user.id, 'facebook')
    end
  end
end
```

### 2.2 On-Demand Refresh

When users request a manual refresh or an extended lookback period:

```ruby
class UserAccountsRefreshJob < ApplicationJob
  queue_as :refresh
  retry_on StandardError, wait: :exponentially_longer, attempts: 3
  
  def perform(user_id, platform_type, days_lookback = 30)
    user = User.find(user_id)
    
    # Process all accounts of the specified platform for this user
    case platform_type
    when 'google'
      user.google_ad_accounts.find_each do |account|
        # Create tracking record
        refresh_job = RefreshJob.create!(
          refreshable_id: account.id,
          refreshable_type: 'GoogleAdAccount',
          user_id: user.id,
          status: 'processing',
          started_at: Time.current,
          days_lookback: days_lookback
        )
        
        begin
          refresh_google_ad_account(account, days_lookback)
          
          refresh_job.update!(
            status: 'completed',
            completed_at: Time.current
          )
        rescue => e
          refresh_job.update!(
            status: 'failed',
            error_message: e.message,
            completed_at: Time.current
          )
        end
      end
    when 'facebook'
      user.facebook_ad_accounts.find_each do |account|
        # Similar implementation for Facebook accounts
      end
    end
    
    # Update prompt cache after all accounts are refreshed
    PromptGenerationJob.perform_later(user_id)
  end
  
  private
  
  def refresh_google_ad_account(account, days_lookback)
    # Initialize Google Ads API client
    client = GoogleAdsApiClient.new(account.oauth_refresh_token)
    
    # Fetch account, campaigns, ad groups, ads data
    fetch_and_store_google_campaigns(client, account, days_lookback)
    
    # Update last refreshed timestamp
    account.update!(last_refreshed_at: Time.current)
  end
  
  # Similar method for Facebook
end
```

## 3. LLM Prompt Generation

### 3.1 Prompt Template Structure

The system will use a template-based approach for generating LLM prompts with the following components:

```ruby
SYSTEM_PROMPT_TEMPLATE = <<~PROMPT
You are Burry AI, a specialized ad performance analysis assistant.
Only respond with information explicitly contained in the context below.
If you're unsure or information is unavailable, use one of these responses:
1. "I don't have {metric} for {time_period}. View in Google Ads: {deep_link}"
2. "This data wasn't in our latest refresh ({timestamp}). Refresh now?"
3. "I can only access {available_metrics} for {available_entities}"

ACCOUNT OVERVIEW:
<%= account_overview %>

TOP CAMPAIGN PERFORMANCE:
<%= top_campaigns %>

TOP AD GROUP PERFORMANCE:
<%= top_ad_groups %>

TOP ADS PERFORMANCE:
<%= top_ads %>

DATA FRESHNESS:
Last updated: <%= last_refreshed_at %>
Data timespan: <%= data_start_date %> to <%= data_end_date %>
PROMPT
```

### 3.2 Prompt Generation Pipeline

The prompt generation process will run as a background job:

```ruby
class PromptGenerationJob < ApplicationJob
  queue_as :prompts
  retry_on StandardError, wait: 30.seconds, attempts: 3
  
  def perform(user_id)
    user = User.find(user_id)
    
    # Create or update prompt for each ad account
    user.google_ad_accounts.each do |account|
      generate_google_account_prompt(user, account)
    end
    
    # Future platforms
    user.facebook_ad_accounts.each do |account|
      generate_facebook_account_prompt(user, account)
    end
  end
  
  private
  
  def generate_google_account_prompt(user, account)
    # Default 30-day lookback period
    end_date = Date.today
    start_date = end_date - 30.days
    
    # Get account overview data
    account_data = fetch_account_overview(account, start_date, end_date)
    
    # Get top 5 campaigns by spend
    top_campaigns_data = fetch_top_campaigns(account, start_date, end_date)
    
    # Get top 5 ad groups per campaign
    top_ad_groups_data = fetch_top_ad_groups(account, top_campaigns_data, start_date, end_date)
    
    # Get top 5 ads per ad group
    top_ads_data = fetch_top_ads(account, top_ad_groups_data, start_date, end_date)
    
    # Render prompt using ERB
    context = {
      account_overview: format_account_overview(account_data),
      top_campaigns: format_campaigns(top_campaigns_data),
      top_ad_groups: format_ad_groups(top_ad_groups_data),
      top_ads: format_ads(top_ads_data),
      last_refreshed_at: account.last_refreshed_at,
      data_start_date: start_date,
      data_end_date: end_date
    }
    
    rendered_prompt = ERB.new(SYSTEM_PROMPT_TEMPLATE).result_with_hash(context)
    
    # Store the rendered prompt
    LlmPrompt.where(user: user, is_active: true).update_all(is_active: false)
    LlmPrompt.create!(
      user: user,
      prompt_template: SYSTEM_PROMPT_TEMPLATE,
      rendered_prompt: rendered_prompt,
      context_data: context,
      generated_at: Time.current,
      expires_at: Time.current + 6.hours,
      is_active: true
    )
  end
  
  # Data fetching methods
  def fetch_account_overview(account, start_date, end_date)
    # Efficient query to get aggregated account metrics
    GoogleDailyMetric.where(
      measurable: account,
      date: start_date..end_date
    ).select(
      'SUM(impressions) as total_impressions',
      'SUM(clicks) as total_clicks',
      'SUM(cost) as total_cost',
      'SUM(conversions) as total_conversions',
      'SUM(conversion_value) as total_conversion_value',
      'AVG(ctr) as avg_ctr',
      'AVG(cpc) as avg_cpc',
      'AVG(cpa) as avg_cpa',
      'AVG(roas) as avg_roas'
    ).first
  end
  
  def fetch_top_campaigns(account, start_date, end_date)
    # Join campaign table with metrics to get top 5 by spend
    account.google_campaigns.joins(:google_daily_metrics)
      .where(google_daily_metrics: { date: start_date..end_date })
      .group('google_campaigns.id')
      .select(
        'google_campaigns.*',
        'SUM(google_daily_metrics.impressions) as total_impressions',
        'SUM(google_daily_metrics.clicks) as total_clicks',
        'SUM(google_daily_metrics.cost) as total_cost',
        'SUM(google_daily_metrics.conversions) as total_conversions',
        'SUM(google_daily_metrics.conversion_value) as total_conversion_value'
      )
      .order('total_cost DESC')
      .limit(5)
  end
  
  # Similar methods for ad groups and ads
end
```

## 4. Hallucination Prevention Implementation

### 4.1 Database Query Validation

To ensure all data is factually accurate, query validation will be implemented:

```ruby
module QueryValidator
  def self.validate_metric_availability(entity_type, metric_name, start_date, end_date)
    valid_metrics = {
      'GoogleAdAccount' => ['impressions', 'clicks', 'cost', 'conversions', 'ctr', 'cpc'],
      'GoogleCampaign' => ['impressions', 'clicks', 'cost', 'conversions', 'ctr', 'cpc'],
      'GoogleAdGroup' => ['impressions', 'clicks', 'cost', 'conversions', 'ctr', 'cpc'],
      'GoogleAd' => ['impressions', 'clicks', 'cost', 'conversions', 'ctr', 'cpc']
    }
    
    return false unless valid_metrics[entity_type]&.include?(metric_name)
    
    # Check if we have data for the requested date range
    case entity_type
    when 'GoogleAdAccount'
      GoogleDailyMetric.where(
        measurable_type: entity_type,
        date: start_date..end_date
      ).exists?
    when 'GoogleCampaign'
      GoogleDailyMetric.where(
        measurable_type: entity_type,
        date: start_date..end_date
      ).exists?
    # Similar checks for other entity types
    end
  end
end
```

### 4.2 Response Validation Pipeline

A post-processing step will be added to validate LLM responses:

```ruby
class ResponseValidator
  CONFIDENCE_THRESHOLD = 0.85
  
  def self.validate(user_query, llm_response, context_data)
    # Check for uncertainty markers in response
    uncertainty_phrases = [
      "I'm not sure", "I don't know", "I'm uncertain",
      "might be", "could be", "possibly", "perhaps"
    ]
    
    has_uncertainty = uncertainty_phrases.any? { |phrase| llm_response.include?(phrase) }
    
    # Check if response mentions metrics not in context
    available_metrics = extract_available_metrics(context_data)
    mentioned_metrics = extract_mentioned_metrics(llm_response)
    
    unknown_metrics = mentioned_metrics - available_metrics
    
    if has_uncertainty || unknown_metrics.any?
      # Generate appropriate fallback response
      generate_fallback_response(user_query, context_data, unknown_metrics)
    else
      # Response passes validation
      llm_response
    end
  end
  
  private
  
  def self.extract_available_metrics(context_data)
    # Extract metrics available in the context data
    # ...
  end
  
  def self.extract_mentioned_metrics(response)
    # Extract metrics mentioned in the response
    # ...
  end
  
  def self.generate_fallback_response(query, context_data, unknown_metrics)
    if unknown_metrics.any?
      "I don't have information about #{unknown_metrics.join(', ')}. " +
      "The metrics I can provide are: #{context_data[:available_metrics].join(', ')}. " +
      "View in Google Ads: #{generate_deep_link(context_data)}"
    else
      "I don't have enough information to answer that question confidently. " +
      "The data was last refreshed at #{context_data[:last_refreshed_at]}. " +
      "Would you like me to refresh the data now?"
    end
  end
end
```

## 5. API Integration

### 5.1 Google Ads API Client

```ruby
class GoogleAdsApiClient
  def initialize(refresh_token)
    @client = GoogleAds::GoogleAdsClient.new do |config|
      config.client_id = ENV['GOOGLE_ADS_CLIENT_ID']
      config.client_secret = ENV['GOOGLE_ADS_CLIENT_SECRET']
      config.refresh_token = refresh_token
      config.login_customer_id = ENV['GOOGLE_ADS_LOGIN_CUSTOMER_ID']
    end
  end
  
  def fetch_campaigns(customer_id, start_date, end_date)
    query = <<~QUERY
      SELECT
        campaign.id,
        campaign.name,
        campaign.status,
        campaign.advertising_channel_type,
        campaign.start_date,
        campaign.end_date,
        metrics.impressions,
        metrics.clicks,
        metrics.cost_micros,
        metrics.conversions,
        metrics.conversions_value
      FROM campaign
      WHERE segments.date BETWEEN '#{start_date}' AND '#{end_date}'
    QUERY
    
    response = @client.service.google_ads.search(customer_id, query)
    
    campaigns = []
    response.each do |row|
      campaigns << {
        id: row.campaign.id,
        name: row.campaign.name,
        status: row.campaign.status.to_s,
        type: row.campaign.advertising_channel_type.to_s,
        start_date: Date.parse(row.campaign.start_date),
        end_date: row.campaign.end_date.present? ? Date.parse(row.campaign.end_date) : nil,
        metrics: {
          impressions: row.metrics.impressions,
          clicks: row.metrics.clicks,
          cost: row.metrics.cost_micros / 1_000_000.0,
          conversions: row.metrics.conversions,
          conversion_value: row.metrics.conversions_value
        }
      }
    end
    
    campaigns
  end
  
  # Similar methods for ad groups and ads
end
```

### 5.2 Slack Integration Client

```ruby
class SlackClient
  def initialize(bot_token)
    @client = Slack::Web::Client.new(token: bot_token)
  end
  
  def send_daily_summary(channel_id, summary_text)
    @client.chat_postMessage(
      channel: channel_id,
      text: summary_text,
      mrkdwn: true
    )
  end
  
  def reply_to_thread(channel_id, thread_ts, response_text)
    @client.chat_postMessage(
      channel: channel_id,
      thread_ts: thread_ts,
      text: response_text,
      mrkdwn: true
    )
  end
end
```

## 6. Monitoring and Feedback System

### 6.1 User Feedback Collection

```ruby
class FeedbackController < ApplicationController
  before_action :authenticate_user!
  
  def create
    interaction = UserInteraction.find(params[:interaction_id])
    
    interaction.update!(
      marked_as_hallucination: params[:is_hallucination],
      user_satisfaction_rating: params[:rating],
      user_feedback: params[:feedback_text]
    )
    
    # If marked as hallucination, trigger analysis
    if params[:is_hallucination]
      HallucinationAnalysisJob.perform_async(interaction.id)
    end
    
    head :ok
  end
end
```

### 6.2 Hallucination Analysis Job

```ruby
class HallucinationAnalysisJob < ApplicationJob
  queue_as :analysis
  
  def perform(interaction_id)
    interaction = UserInteraction.find(interaction_id)
    
    # Log hallucination instance
    Rails.logger.warn("Hallucination detected: #{interaction.id}")
    
    # Store hallucination details for later analysis
    HallucinationReport.create!(
      user_interaction: interaction,
      query_type: classify_query_type(interaction.user_query),
      response_content: interaction.system_response,
      context_data: interaction.context_data,
      detected_at: Time.current
    )
    
    # Notify admins if high rate of hallucinations
    hallucination_rate = calculate_recent_hallucination_rate
    if hallucination_rate > 0.05 # 5% threshold
      AdminNotifier.hallucination_alert(hallucination_rate).deliver_later
    end
  end
  
  private
  
  def classify_query_type(query)
    # Classify query into categories for analysis
    # ...
  end
  
  def calculate_recent_hallucination_rate
    total = UserInteraction.where('created_at > ?', 24.hours.ago).count
    hallucinations = UserInteraction.where(
      marked_as_hallucination: true,
      created_at: 24.hours.ago..Time.current
    ).count
    
    total > 0 ? hallucinations.to_f / total : 0
  end
end
```

## 7. Testing Strategy

### 7.1 Database Schema Tests

```ruby
RSpec.describe "Database Schema" do
  context "Metrics data integrity" do
    it "prevents duplicate daily metrics for the same entity and date" do
      campaign = create(:google_campaign)
      metric = create(:google_daily_metric, measurable: campaign, date: Date.today)
      
      duplicate = build(:google_daily_metric, measurable: campaign, date: Date.today)
      expect(duplicate).not_to be_valid
    end
    
    it "cascades deletes from parent entities to metrics" do
      campaign = create(:google_campaign)
      metric = create(:google_daily_metric, measurable: campaign, date: Date.today)
      
      expect { campaign.destroy }.to change { GoogleDailyMetric.count }.by(-1)
    end
  end
end
```

### 7.2 LLM Response Validation Tests

```ruby
RSpec.describe ResponseValidator do
  describe ".validate" do
    it "detects and replaces responses with uncertainty markers" do
      response = "I think the campaign had about 1000 impressions, but I'm not sure."
      context = { available_metrics: ['impressions', 'clicks'], last_refreshed_at: Time.current }
      
      validated = ResponseValidator.validate("How many impressions?", response, context)
      expect(validated).to include("I don't have enough information")
    end
    
    it "detects metrics not present in context data" do
      response = "The campaign had 1000 impressions and 50 conversions."
      context = { available_metrics: ['impressions', 'clicks'], last_refreshed_at: Time.current }
      
      validated = ResponseValidator.validate("How did the campaign perform?", response, context)
      expect(validated).to include("I don't have information about conversions")
    end
  end
end
```

## 8. LLM Provider Selection

### 8.1 LLM Provider Interface

To enable flexibility in LLM selection, we'll implement an adapter pattern:

```ruby
# Base LLM provider interface
class LlmProvider
  def initialize(api_key)
    @api_key = api_key
  end
  
  def generate_response(prompt, context)
    raise NotImplementedError, "Subclasses must implement this method"
  end
  
  def calculate_cost(prompt_tokens, response_tokens)
    raise NotImplementedError, "Subclasses must implement this method"
  end
end

# Implementation for OpenAI
class OpenAiProvider < LlmProvider
  def generate_response(prompt, context = {})
    client = OpenAI::Client.new(access_token: @api_key)
    
    response = client.chat(
      parameters: {
        model: context[:model] || "gpt-4o",
        messages: [
          { role: "system", content: prompt },
          { role: "user", content: context[:user_query] }
        ],
        temperature: context[:temperature] || 0.2,
        max_tokens: context[:max_tokens] || 1000
      }
    )
    
    response.dig("choices", 0, "message", "content")
  end
  
  def calculate_cost(prompt_tokens, response_tokens)
    # Cost calculation for GPT-4o
    (prompt_tokens / 1000.0) * 0.01 + (response_tokens / 1000.0) * 0.03
  end
end

# Implementation for Claude
class ClaudeProvider < LlmProvider
  def generate_response(prompt, context = {})
    client = Anthropic::Client.new(api_key: @api_key)
    
    response = client.messages.create(
      model: context[:model] || "claude-3-5-sonnet",
      system: prompt,
      messages: [{ role: "user", content: context[:user_query] }],
      temperature: context[:temperature] || 0.2,
      max_tokens: context[:max_tokens] || 1000
    )
    
    response.content.first.text
  end
  
  def calculate_cost(prompt_tokens, response_tokens)
    # Cost calculation for Claude 3.5 Sonnet
    (prompt_tokens / 1000.0) * 0.003 + (response_tokens / 1000.0) * 0.015
  end
end

# Implementation for Gemini
class GeminiProvider < LlmProvider
  def generate_response(prompt, context = {})
    client = Google::Cloud::VertexAI.gemini_client(
      credentials: @api_key, 
      endpoint: "us-central1-aiplatform.googleapis.com"
    )
    
    response = client.generate_content(
      "gemini-1.5-pro",
      [
        { text: "#{prompt}\n\nUser: #{context[:user_query]}" }
      ],
      generation_config: {
        temperature: context[:temperature] || 0.2,
        max_output_tokens: context[:max_tokens] || 1000
      }
    )
    
    response.candidates.first.content.parts.first.text
  end
  
  def calculate_cost(prompt_tokens, response_tokens)
    # Cost calculation for Gemini 1.5 Pro
    (prompt_tokens / 1000.0) * 0.0025 + (response_tokens / 1000.0) * 0.0075
  end
end
```

### 8.2 Provider Selection Strategy

We'll implement a provider selection strategy based on:

1. **Cost optimization**: Choose provider based on token usage and budget constraints
2. **Performance metrics**: Track response quality and speed for each provider
3. **Feature requirements**: Select providers based on specific capabilities needed

```ruby
class LlmProviderSelector
  def self.select_provider(user_query, context_size, user_preferences = {})
    # Default weights for selection criteria
    weights = {
      cost: 0.4,
      performance: 0.4,
      features: 0.2
    }.merge(user_preferences[:weights] || {})
    
    # Calculate estimated token count
    estimated_tokens = estimate_token_count(user_query, context_size)
    
    # Get provider scores
    providers = {
      openai: score_provider('openai', estimated_tokens, weights),
      claude: score_provider('claude', estimated_tokens, weights),
      gemini: score_provider('gemini', estimated_tokens, weights)
    }
    
    # Return highest scoring provider
    providers.max_by { |_, score| score }.first
  end
  
  private
  
  def self.score_provider(provider, token_count, weights)
    performance_score = ProviderPerformanceMetric.average_score(provider)
    cost_score = calculate_cost_score(provider, token_count)
    feature_score = calculate_feature_score(provider)
    
    (performance_score * weights[:performance]) +
    (cost_score * weights[:cost]) +
    (feature_score * weights[:features])
  end
  
  def self.calculate_cost_score(provider, token_count)
    # Calculate normalized cost score (1.0 = lowest cost)
    provider_costs = {
      'openai' => (token_count / 1000.0) * 0.04, # GPT-4o combined rate
      'claude' => (token_count / 1000.0) * 0.018, # Claude 3.5 Sonnet combined rate
      'gemini' => (token_count / 1000.0) * 0.01 # Gemini 1.5 Pro combined rate
    }
    
    min_cost = provider_costs.values.min
    max_cost = provider_costs.values.max
    
    # Normalize: 1.0 for lowest cost, 0.0 for highest
    cost_range = max_cost - min_cost
    if cost_range.zero?
      1.0
    else
      1.0 - ((provider_costs[provider] - min_cost) / cost_range)
    end
  end
  
  def self.calculate_feature_score(provider)
    # Feature availability scoring
    # 1.0 = all required features available, 0.0 = no required features
    provider_features = {
      'openai' => {
        function_calling: true,
        json_mode: true,
        vision: true
      },
      'claude' => {
        function_calling: true,
        json_mode: true,
        vision: true
      },
      'gemini' => {
        function_calling: true,
        json_mode: true,
        vision: true
      }
    }
    
    # Calculate score based on needed features
    features_needed = [:function_calling, :json_mode]
    features_available = features_needed.count { |f| provider_features[provider][f] }
    
    features_available.to_f / features_needed.size
  end
end
```

## 9. Deployment Architecture

### 9.1 Infrastructure

The application will be deployed using the following infrastructure:

- **Web Application**: Rails application deployed on Heroku
- **Background Jobs**: GoodJob (uses PostgreSQL, avoids Redis dependency)
- **Database**: Managed PostgreSQL instance (Heroku PostgreSQL)
- **Caching**: PostgreSQL-based caching strategy with pgbouncer
- **LLM Integration**: Adapter-based approach for OpenAI/Claude/Gemini

### 9.2 Scaling Considerations

- Database partitioning for metrics tables (by date ranges)
- Read replicas for analytics queries
- Job queue priority settings via GoodJob:
  - High: User-initiated refreshes
  - Medium: Daily summaries
  - Low: Routine refreshes and analytics
- Caching strategy for prompt templates and frequency-limited responses
