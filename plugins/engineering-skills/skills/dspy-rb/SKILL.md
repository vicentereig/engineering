---
name: dspy-rb
description: Build type-safe LLM applications with DSPy.rb - Ruby's programmatic prompt framework with signatures, modules, agents, and optimization
---

# DSPy.rb

> Build LLM apps like you build software. Type-safe, modular, testable.

DSPy.rb brings software engineering best practices to LLM development. Instead of tweaking prompts, you define what you want with Ruby types and let DSPy handle the rest.

## Overview

DSPy.rb is a Ruby framework for building language model applications with programmatic prompts. It provides:

- **Type-safe signatures** - Define inputs/outputs with Sorbet types
- **Modular components** - Compose and reuse LLM logic
- **Automatic optimization** - Use data to improve prompts, not guesswork
- **Production-ready** - Built-in observability, testing, and error handling

## Core Concepts

### 1. Signatures

Define interfaces between your app and LLMs using Ruby types:

```ruby
class EmailClassifier < DSPy::Signature
  description "Classify customer support emails by category and priority"

  class Priority < T::Enum
    enums do
      Low = new('low')
      Medium = new('medium')
      High = new('high')
      Urgent = new('urgent')
    end
  end

  input do
    const :email_content, String
    const :sender, String
  end

  output do
    const :category, String
    const :priority, Priority  # Type-safe enum with defined values
    const :confidence, Float
  end
end
```

### 2. Modules

Build complex workflows from simple building blocks:

- **Predict** - Basic LLM calls with signatures
- **ChainOfThought** - Step-by-step reasoning
- **ReAct** - Tool-using agents
- **CodeAct** - Dynamic code generation agents (install the `dspy-code_act` gem)

#### Lifecycle callbacks
Rails-style lifecycle hooks ship with every `DSPy::Module`, so you can wrap `forward` without touching instrumentation:

- **`before`** – runs ahead of `forward` for setup (metrics, context loading)
- **`around`** – wraps `forward`, calls `yield`, and lets you pair setup/teardown logic
- **`after`** – fires after `forward` returns for cleanup or persistence

### 3. Tools & Toolsets
Create type-safe tools for agents with comprehensive Sorbet support:

```ruby
# Enum-based tool with automatic type conversion
class CalculatorTool < DSPy::Tools::Base
  tool_name 'calculator'
  tool_description 'Performs arithmetic operations with type-safe enum inputs'

  class Operation < T::Enum
    enums do
      Add = new('add')
      Subtract = new('subtract')
      Multiply = new('multiply')
      Divide = new('divide')
    end
  end

  sig { params(operation: Operation, num1: Float, num2: Float).returns(T.any(Float, String)) }
  def call(operation:, num1:, num2:)
    case operation
    when Operation::Add then num1 + num2
    when Operation::Subtract then num1 - num2
    when Operation::Multiply then num1 * num2
    when Operation::Divide
      return "Error: Division by zero" if num2 == 0
      num1 / num2
    end
  end
end

# Multi-tool toolset with rich types
class DataToolset < DSPy::Tools::Toolset
  toolset_name "data_processing"

  class Format < T::Enum
    enums do
      JSON = new('json')
      CSV = new('csv')
      XML = new('xml')
    end
  end

  class ProcessingConfig < T::Struct
    const :max_rows, Integer, default: 1000
    const :include_headers, T::Boolean, default: true
    const :encoding, String, default: 'utf-8'
  end

  tool :convert, description: "Convert data between formats"
  tool :validate, description: "Validate data structure"

  sig { params(data: String, from: Format, to: Format, config: T.nilable(ProcessingConfig)).returns(String) }
  def convert(data:, from:, to:, config: nil)
    config ||= ProcessingConfig.new
    "Converted from #{from.serialize} to #{to.serialize} with config: #{config.inspect}"
  end

  sig { params(data: String, format: Format).returns(T::Hash[String, T.any(String, Integer, T::Boolean)]) }
  def validate(data:, format:)
    {
      valid: true,
      format: format.serialize,
      row_count: 42,
      message: "Data validation passed"
    }
  end
end
```

### 4. Type System & Discriminators
DSPy.rb uses sophisticated type discrimination for complex data structures:

- **Automatic `_type` field injection** - DSPy adds discriminator fields to structs for type safety
- **Union type support** - T.any() types automatically disambiguated by `_type`
- **Reserved field name** - Avoid defining your own `_type` fields in structs
- **Recursive filtering** - `_type` fields filtered during deserialization at all nesting levels

### 5. Optimization
Improve accuracy with real data:

- **MIPROv2** - Advanced multi-prompt optimization with bootstrap sampling and Bayesian optimization
- **GEPA (Genetic-Pareto Reflective Prompt Evolution)** - Reflection-driven instruction rewrite loop with feedback maps, experiment tracking, and telemetry
- **Evaluation** - Comprehensive framework with built-in and custom metrics, error handling, and batch processing

## Quick Start

```ruby
# Install
gem 'dspy'

# Configure
DSPy.configure do |c|
  c.lm = DSPy::LM.new('openai/gpt-4o-mini', api_key: ENV['OPENAI_API_KEY'])
  # or use Ollama for local models
  # c.lm = DSPy::LM.new('ollama/llama3.2')
end

# Define a task
class SentimentAnalysis < DSPy::Signature
  description "Analyze sentiment of text"

  input do
    const :text, String
  end

  output do
    const :sentiment, String  # positive, negative, neutral
    const :score, Float       # 0.0 to 1.0
  end
end

# Use it
analyzer = DSPy::Predict.new(SentimentAnalysis)
result = analyzer.call(text: "This product is amazing!")
puts result.sentiment  # => "positive"
puts result.score      # => 0.92
```

## Provider Adapter Gems

Two strategies for connecting to LLM providers:

### Per-provider adapters (direct SDK access)

```ruby
# Gemfile
gem 'dspy'
gem 'dspy-openai'    # OpenAI, OpenRouter, Ollama
gem 'dspy-anthropic' # Claude
gem 'dspy-gemini'    # Gemini
```

Each adapter gem pulls in the official SDK (`openai`, `anthropic`, `gemini-ai`).

### Unified adapter via RubyLLM (recommended for multi-provider)

```ruby
# Gemfile
gem 'dspy'
gem 'dspy-ruby_llm'  # Routes to any provider via ruby_llm
gem 'ruby_llm'
```

RubyLLM handles provider routing based on the model name. Use the `ruby_llm/` prefix:

```ruby
DSPy.configure do |c|
  c.lm = DSPy::LM.new('ruby_llm/gemini-2.5-flash', structured_outputs: true)
  # c.lm = DSPy::LM.new('ruby_llm/claude-sonnet-4-20250514', structured_outputs: true)
  # c.lm = DSPy::LM.new('ruby_llm/gpt-4o-mini', structured_outputs: true)
end
```

## Rails Integration

### Directory Structure

Organize DSPy components using Rails conventions:

```
app/
  entities/          # T::Struct types shared across signatures
  signatures/        # DSPy::Signature definitions
  tools/             # DSPy::Tools::Base implementations
    concerns/        # Shared tool behaviors (error handling, etc.)
  modules/           # DSPy::Module orchestrators
  services/          # Plain Ruby services that compose DSPy modules
config/
  initializers/
    dspy.rb          # DSPy + provider configuration
    feature_flags.rb # Model selection per role
spec/
  signatures/        # Schema validation tests
  tools/             # Tool unit tests
  modules/           # Integration tests with VCR
  vcr_cassettes/     # Recorded HTTP interactions
```

### Initializer

```ruby
# config/initializers/dspy.rb
Rails.application.config.after_initialize do
  next if Rails.env.test? && ENV["DSPY_ENABLE_IN_TEST"].blank?

  RubyLLM.configure do |config|
    config.gemini_api_key = ENV["GEMINI_API_KEY"] if ENV["GEMINI_API_KEY"].present?
    config.anthropic_api_key = ENV["ANTHROPIC_API_KEY"] if ENV["ANTHROPIC_API_KEY"].present?
    config.openai_api_key = ENV["OPENAI_API_KEY"] if ENV["OPENAI_API_KEY"].present?
  end

  model = ENV.fetch("DSPY_MODEL", "ruby_llm/gemini-2.5-flash")
  DSPy.configure do |config|
    config.lm = DSPy::LM.new(model, structured_outputs: true)
    config.logger = Rails.logger
  end

  # Langfuse observability (optional)
  if ENV["LANGFUSE_PUBLIC_KEY"].present? && ENV["LANGFUSE_SECRET_KEY"].present?
    DSPy::Observability.configure!
  end
end
```

### Feature-Flagged Model Selection

Use different models for different roles (fast/cheap for classification, powerful for synthesis):

```ruby
# config/initializers/feature_flags.rb
module FeatureFlags
  SELECTOR_MODEL = ENV.fetch("DSPY_SELECTOR_MODEL", "ruby_llm/gemini-2.5-flash-lite")
  SYNTHESIZER_MODEL = ENV.fetch("DSPY_SYNTHESIZER_MODEL", "ruby_llm/gemini-2.5-flash")
end
```

Then override per-tool or per-predictor:

```ruby
class ClassifyTool < DSPy::Tools::Base
  def call(query:)
    predictor = DSPy::Predict.new(ClassifyQuery)
    predictor.configure { |c| c.lm = DSPy::LM.new(FeatureFlags::SELECTOR_MODEL, structured_outputs: true) }
    predictor.call(query: query)
  end
end
```

## Schema-Driven Signatures

**Prefer typed schemas over string descriptions.** Let the type system communicate structure to the LLM rather than prose in the signature description.

### Entities as Shared Types

Define reusable `T::Struct` and `T::Enum` types in `app/entities/` and reference them across signatures:

```ruby
# app/entities/search_strategy.rb
class SearchStrategy < T::Enum
  enums do
    SingleSearch = new("single_search")
    DateDecomposition = new("date_decomposition")
  end
end

# app/entities/date_window.rb
class DateWindow < T::Struct
  const :date_from, String
  const :date_to, String
end

# app/entities/scored_item.rb
class ScoredItem < T::Struct
  const :id, String
  const :score, Float, description: "Relevance score 0.0-1.0"
  const :verdict, String, description: "relevant, maybe, or irrelevant"
  const :reason, String, default: ""
end
```

### Signatures Reference Entities

```ruby
# app/signatures/classify_research_query.rb
class ClassifyResearchQuery < DSPy::Signature
  description "Classify a research query and produce the search plan."

  input do
    const :query, String
    const :available_sources, T::Array[Source]
  end

  output do
    const :requires_exhaustive_research, T::Boolean
    const :terminology_variants, T::Array[String]
    const :target_sources, T::Array[Source]
    const :date_from, T.nilable(String)
    const :date_to, T.nilable(String)
    const :search_strategy, SearchStrategy
    const :date_windows, T::Array[DateWindow], default: []
    const :exclusion_terms, T::Array[String], default: []
  end
end
```

### Schema vs Description: When to Use Each

**Use schemas (T::Struct/T::Enum)** for:
- Multi-field outputs with specific types
- Enums with defined values the LLM must pick from
- Nested structures, arrays of typed objects
- Outputs consumed by code (not displayed to users)

**Use string descriptions** for:
- Simple single-field outputs where the type is `String`
- Natural language generation (summaries, answers)
- Fields where constraint guidance helps (e.g., `description: "YYYY-MM-DD format"`)

**Rule of thumb**: If you'd write a `case` statement on the output, it should be a `T::Enum`. If you'd call `.each` on it, it should be `T::Array[SomeStruct]`.

## Tool Patterns

### Tools That Wrap Predictions

A common pattern: tools encapsulate a DSPy prediction, adding error handling, model selection, and serialization:

```ruby
# app/tools/rerank_tool.rb
class RerankTool < DSPy::Tools::Base
  tool_name "rerank"
  tool_description "Score and rank search results by relevance"

  MAX_ITEMS = 200
  MIN_ITEMS_FOR_LLM = 5

  sig { params(query: String, items: T::Array[T::Hash[Symbol, T.untyped]]).returns(T::Hash[Symbol, T.untyped]) }
  def call(query:, items: [])
    # Short-circuit: skip LLM for small sets
    return { scored_items: items, reranked: false } if items.size < MIN_ITEMS_FOR_LLM

    # Cap to prevent token overflow
    capped_items = items.first(MAX_ITEMS)

    predictor = DSPy::Predict.new(RerankSignature)
    predictor.configure { |c| c.lm = DSPy::LM.new(FeatureFlags::SYNTHESIZER_MODEL, structured_outputs: true) }

    result = predictor.call(query: query, items: capped_items)
    { scored_items: result.scored_items, reranked: true }
  rescue => e
    Rails.logger.warn "[RerankTool] LLM rerank failed: #{e.message}"
    { error: "Rerank failed: #{e.message}", scored_items: items, reranked: false }
  end
end
```

**Key patterns:**
- Short-circuit LLM calls when unnecessary (small data, trivial cases)
- Cap input size to prevent token overflow
- Per-tool model selection via `configure`
- Graceful error handling with fallback data

### Error Handling Concern

```ruby
# app/tools/concerns/error_handling.rb
module ErrorHandling
  extend ActiveSupport::Concern

  private

  def safe_predict(signature_class, **inputs)
    predictor = DSPy::Predict.new(signature_class)
    yield predictor if block_given?
    predictor.call(**inputs)
  rescue Faraday::Error, Net::HTTPError => e
    Rails.logger.error "[#{self.class.name}] API error: #{e.message}"
    nil
  rescue JSON::ParserError => e
    Rails.logger.error "[#{self.class.name}] Invalid LLM output: #{e.message}"
    nil
  end
end
```

## Module Composition

### Custom Tool-Calling Module

Build a module that selects and dispatches tools via LLM reasoning:

```ruby
# app/modules/tool_selector.rb
class ToolSelector < DSPy::Module
  class SelectionSignature < DSPy::Signature
    description "Select the best tool for the current step."

    input do
      const :query, String
      const :context, String
      const :available_tools, T::Array[T::Hash[Symbol, T.untyped]]
    end

    output do
      const :reasoning, String
      const :tool_name, String
      const :tool_args, String, description: "JSON-encoded arguments"
    end
  end

  def initialize(tools:)
    super()
    @tools = tools
    @tools_by_name = tools.index_by { |t| t.name.downcase }
    @predictor = DSPy::Predict.new(SelectionSignature)
  end

  def forward(query:, context:)
    schemas = @tools.map { |t| { name: t.name, description: t.description, parameters: t.class.call_schema_object } }

    result = @predictor.call(query: query, context: context, available_tools: schemas)
    tool = @tools_by_name[result.tool_name.downcase]
    args = JSON.parse(result.tool_args, symbolize_names: true)

    { tool_name: result.tool_name, tool_result: tool.dynamic_call(args), reasoning: result.reasoning }
  end
end
```
## Observability

### Tracing with DSPy::Context

Wrap operations in spans for Langfuse/OpenTelemetry visibility:

```ruby
result = DSPy::Context.with_span(
  operation: "tool_selector.select",
  "dspy.module" => "ToolSelector",
  "tool_selector.tools" => tool_names.join(",")
) do
  @predictor.call(query: query, context: context, available_tools: schemas)
end
```

### Setup for Langfuse

```ruby
# Gemfile
gem 'dspy-o11y'
gem 'dspy-o11y-langfuse'

# .env
LANGFUSE_PUBLIC_KEY=pk-...
LANGFUSE_SECRET_KEY=sk-...
DSPY_TELEMETRY_BATCH_SIZE=5
```

Every `DSPy::Predict`, `DSPy::ReAct`, and tool call is automatically traced when observability is configured.

## Testing

### VCR Setup for Rails

```ruby
# spec/rails_helper.rb (or spec/support/dspy.rb)
VCR.configure do |config|
  config.cassette_library_dir = "spec/vcr_cassettes"
  config.hook_into :webmock
  config.configure_rspec_metadata!
  config.filter_sensitive_data('<GEMINI_API_KEY>') { ENV['GEMINI_API_KEY'] }
  config.filter_sensitive_data('<OPENAI_API_KEY>') { ENV['OPENAI_API_KEY'] }
  config.filter_sensitive_data('<ANTHROPIC_API_KEY>') { ENV['ANTHROPIC_API_KEY'] }
end
```

### Signature Schema Tests

Test that signatures produce valid schemas without calling any LLM:

```ruby
# spec/signatures/classify_research_query_spec.rb
RSpec.describe ClassifyResearchQuery do
  it "has required input fields" do
    schema = described_class.input_json_schema
    expect(schema[:required]).to include("query")
  end

  it "has typed output fields" do
    schema = described_class.output_json_schema
    expect(schema[:properties]).to have_key(:search_strategy)
    expect(schema[:properties]).to have_key(:terminology_variants)
  end
end
```

### Tool Tests with Mocked Predictions

```ruby
# spec/tools/rerank_tool_spec.rb
RSpec.describe RerankTool do
  let(:tool) { described_class.new }

  it "skips LLM for small result sets" do
    expect(DSPy::Predict).not_to receive(:new)
    result = tool.call(query: "test", items: [{ id: "1" }])
    expect(result[:reranked]).to be false
  end

  it "calls LLM for large result sets", :vcr do
    items = 10.times.map { |i| { id: i.to_s, title: "Item #{i}" } }
    result = tool.call(query: "relevant items", items: items)
    expect(result[:reranked]).to be true
    expect(result[:scored_items]).to be_an(Array)
  end
end
```

### Integration Tests

```ruby
# spec/modules/research_navigator_spec.rb
RSpec.describe ResearchNavigator, :vcr do
  let(:tools) { [SearchTool.new, FetchTool.new, FinishTool.new] }
  let(:navigator) { described_class.new(tools: tools) }

  it "completes research within budget" do
    result = navigator.call(query: "Find recent regulations on data privacy")
    expect(result[:answer]).to be_present
    expect(result[:iterations]).to be <= ResearchNavigator::MAX_ITERATIONS
  end
end
```

## Key URLs

- Homepage: https://oss.vicente.services/dspy.rb/
- GitHub: https://github.com/vicentereig/dspy.rb
- Documentation: https://oss.vicente.services/dspy.rb/getting-started/

## Guidelines for Claude

When helping users with DSPy.rb:

1. **Schema over prose** - Define output structure with `T::Struct` and `T::Enum` types, not string descriptions
2. **Entities in `app/entities/`** - Extract shared types so signatures stay thin
3. **Per-tool model selection** - Use `predictor.configure { |c| c.lm = ... }` to pick the right model per task
4. **Short-circuit LLM calls** - Skip the LLM for trivial cases (small data, cached results)
5. **Cap input sizes** - Prevent token overflow by limiting array sizes before sending to LLM
6. **Test schemas without LLM** - Validate `input_json_schema` and `output_json_schema` in unit tests
7. **VCR for integration tests** - Record real HTTP interactions, never mock LLM responses by hand
8. **Trace with spans** - Wrap tool calls in `DSPy::Context.with_span` for observability
9. **Graceful degradation** - Always rescue LLM errors and return fallback data

### Signature Best Practices

**Keep description concise** - The signature `description` should state the goal, not the field details:

```ruby
# Good - concise goal
class ParseOutline < DSPy::Signature
  description 'Extract block-level structure from HTML as a flat list of skeleton sections.'

  input do
    const :html, String, description: 'Raw HTML to parse'
  end

  output do
    const :sections, T::Array[Section], description: 'Block elements: headings, paragraphs, code blocks, lists'
  end
end

# Bad - putting field docs in signature description
class ParseOutline < DSPy::Signature
  description <<~DESC
    Extract outline from HTML.

    Return sections with:
    - node_type: The type of element
    - text: For headings, the text content
    - level: For headings, 1-6
    ...
  DESC
end
```

**Use defaults over nilable arrays** - For OpenAI structured outputs compatibility:

```ruby
# Good - works with OpenAI structured outputs
class ASTNode < T::Struct
  const :children, T::Array[ASTNode], default: []
end

# Bad - causes schema issues with OpenAI
class ASTNode < T::Struct
  const :children, T.nilable(T::Array[ASTNode])
end
```

### Recursive Types with `$defs`

DSPy.rb supports recursive types in structured outputs using JSON Schema `$defs`:

```ruby
class TreeNode < T::Struct
  const :value, String
  const :children, T::Array[TreeNode], default: []  # Self-reference
end

class DocumentAST < DSPy::Signature
  description 'Parse document into tree structure'

  output do
    const :root, TreeNode
  end
end
```

The schema generator automatically creates `#/$defs/TreeNode` references for recursive types, compatible with OpenAI and Gemini structured outputs.

### Field Descriptions for T::Struct

DSPy.rb extends T::Struct to support field-level `description:` kwargs that flow to JSON Schema:

```ruby
class ASTNode < T::Struct
  const :node_type, NodeType, description: 'The type of node (heading, paragraph, etc.)'
  const :text, String, default: "", description: 'Text content of the node'
  const :level, Integer, default: 0  # No description - field is self-explanatory
  const :children, T::Array[ASTNode], default: []
end

# Access descriptions programmatically
ASTNode.field_descriptions[:node_type]  # => "The type of node (heading, paragraph, etc.)"
ASTNode.field_descriptions[:text]       # => "Text content of the node"
ASTNode.field_descriptions[:level]      # => nil (no description)
```

**When to use field descriptions**:
- Complex field semantics not obvious from the type
- Enum-like strings with specific allowed values
- Fields with constraints (e.g., "1-6 for heading levels")
- Nested structs where the purpose isn't clear from the name

**When to skip descriptions**:
- Self-explanatory fields like `name`, `id`, `url`
- Fields where the type tells the story (e.g., `T::Boolean` for flags)

### Hierarchical Parsing for Complex Documents

For complex documents that may exceed token limits, consider two-phase parsing:

1. **Phase 1 - Outline**: Extract skeleton structure (block types, headings)
2. **Phase 2 - Fill**: Parse each section in detail

This avoids max_tokens limits and produces more complete output.

## Version

Current: 0.34.3
