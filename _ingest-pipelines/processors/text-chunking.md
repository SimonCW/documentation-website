---
layout: default
title: Text chunking
parent: Ingest processors
nav_order: 250
---

# Text chunking processor

The `text_chunking` processor splits a long document into shorter passages. The processor supports the following algorithms for text splitting:

- [`fixed_token_length`](#fixed-token-length-algorithm): Splits text into passages of the specified size.
- [`delimiter`](#delimiter-algorithm): Splits text into passages on a delimiter. 

The following is the syntax for the `text_chunking` processor:

```json
{
  "text_chunking": {
    "field_map": {
      "<input_field>": "<output_field>"
    },
    "algorithm": {
      "<name>": "<parameters>"
    }
  }
}
```

## Configuration parameters

The following table lists the required and optional parameters for the `text_chunking` processor.

| Parameter  | Data type | Required/Optional  | Description  |
|:---|:---|:---|:---|
| `field_map` | Object | Required	 | Contains key-value pairs that specify the mapping of a text field to the output field.	  |
| `field_map.<input_field>`	  | String	| Required	 | The name of the field from which to obtain text for generating chunked passages.	                                   |
| `field_map.<output_field>`	 | String	    | Required	 | The name of the field in which to store the chunked results.	|
| `algorithm`	| Object	    | Required	 | Contains at most one key-value pair that specifies the chunking algorithm and parameters. |
| `algorithm.<name>` | String	    | Optional	 | The name of the chunking algorithm. Valid values are [`fixed_token_length`](#fixed-token-length-algorithm) or [`delimiter`](#delimiter-algorithm). Default is `fixed_token_length`.	|
| `algorithm.<parameters>`	   | Object	    | Optional	 | The parameters for the chunking algorithm. By default, contains the default parameters of the `fixed_token_length` algorithm.	 |
| `description`	              | String	    | Optional	 | A brief description of the processor. |
| `tag`	| String	    | Optional	 | An identifier tag for the processor. Useful when debugging in order to distinguish between processors of the same type.	|

### Fixed token length algorithm

The following table lists the optional parameters for the `fixed_token_length` algorithm.

| Parameter  | Data type | Required/Optional  | Description  |
|:---|:---|:---|:---|
| `token_limit`	     | Integer	   | Optional	 | The token limit for chunking algorithms. Valid values are integers of at least `1`. Default is `384`.	                                                  |
| `tokenizer`	       | String	    | Optional	 | The [word tokenizer]({{site.url}}{{site.baseurl}}/analyzers/tokenizers/index/#word-tokenizers) name. Default is `standard`.	 |
| `overlap_rate`	    | String	    | Optional	 | The degree of overlap in the token algorithm. Valid values are floats between `0` and `0.5`, inclusive. Default is `0`.	                                              |
| `max_chunk_limit`	 | Integer	   | Optional	 | The chunk limit for chunking algorithms. Default is 100. To disable this parameter, set it to `-1`.	|

The default value of `token_limit` is `384` so that output passages don't exceed the token limit constraint of the downstream text embedding models. For [OpenSearch-supported pretrained models]({{site.url}}{{site.baseurl}}/ml-commons-plugin/pretrained-models/#supported-pretrained-models), like `msmarco-distilbert-base-tas-b` and `opensearch-neural-sparse-encoding-v1`, the input token limit is `512`. The `standard` tokenizer tokenizes text into words. According to [OpenAI](https://platform.openai.com/docs/introduction), 1 token equals approximately 0.75 words of English text. The default token limit is calculated as 512 * 0.75 = 384.
{: .note}

You can set the `overlap_rate` to a decimal percentage value in the 0--0.5 range, inclusive. Per [Amazon Bedrock](https://aws.amazon.com/blogs/aws/knowledge-bases-now-delivers-fully-managed-rag-experience-in-amazon-bedrock/), we recommend setting this parameter to a value of 0–0.2 to improve accuracy.
{: .note}

The `max_chunk_limit` parameter limits the number of chunked passages. If the number of passages generated by the processor exceeds the limit, the algorithm will return an exception, prompting you to either increase or disable the limit.
{: .note}

### Delimiter algorithm

The following table lists the optional parameters for the `delimiter` algorithm.

| Parameter  | Data type | Required/Optional  | Description  |
|:---|:---|:---|:---|
| `delimiter`	| String	    | Optional	 | A string delimiter used to split text. You can set the `delimiter` to any string, for example, `\n` (split text into paragraphs on a new line) or `.` (split text into sentences). Default is `\n\n` (split text into paragraphs on two new line characters). |
| `max_chunk_limit`	 | Integer	   | Optional	 | The chunk limit for chunking algorithms. Default is `100`. To disable this parameter, set it to `-1`.	 |

The `max_chunk_limit` parameter limits the number of chunked passages. If the number of passages generated by the processor exceeds the limit, the algorithm will return an exception, prompting you to either increase or disable the limit.
{: .note}

## Using the processor

Follow these steps to use the processor in a pipeline. You can specify the chunking algorithm when creating the processor. If you don't provide an algorithm name, the chunking processor will use the default `fixed_token_length` algorithm along with all its default parameters.

**Step 1: Create a pipeline**

The following example request creates an ingest pipeline that converts the text in the `passage_text` field into chunked passages, which will be stored in the `passage_chunk` field:

```json
PUT _ingest/pipeline/text-chunking-ingest-pipeline
{
  "description": "A text chunking ingest pipeline",
  "processors": [
    {
      "text_chunking": {
        "algorithm": {
          "fixed_token_length": {
            "token_limit": 10,
            "overlap_rate": 0.2,
            "tokenizer": "standard"
          }
        },
        "field_map": {
          "passage_text": "passage_chunk"
        }
      }
    }
  ]
}
```
{% include copy-curl.html %}

**Step 2 (Optional): Test the pipeline**

It is recommended that you test your pipeline before ingesting documents.
{: .tip}

To test the pipeline, run the following query:

```json
POST _ingest/pipeline/text-chunking-ingest-pipeline/_simulate
{
  "docs": [
    {
      "_index": "testindex",
      "_id": "1",
      "_source":{
         "passage_text": "This is an example document to be chunked. The document contains a single paragraph, two sentences and 24 tokens by standard tokenizer in OpenSearch."
      }
    }
  ]
}
```
{% include copy-curl.html %}

#### Response

The response confirms that, in addition to the `passage_text` field, the processor has generated chunking results in the `passage_chunk` field. The processor split the paragraph into 10-word chunks. Because of the `overlap` setting of 0.2, the last 2 words of a chunk are duplicated in the following chunk:

```json
{
  "docs": [
    {
      "doc": {
        "_index": "testindex",
        "_id": "1",
        "_source": {
          "passage_text": "This is an example document to be chunked. The document contains a single paragraph, two sentences and 24 tokens by standard tokenizer in OpenSearch.",
          "passage_chunk": [
            "This is an example document to be chunked. The document ",
            "The document contains a single paragraph, two sentences and 24 ",
            "and 24 tokens by standard tokenizer in OpenSearch."
          ]
        },
        "_ingest": {
          "timestamp": "2024-03-20T02:55:25.642366Z"
        }
      }
    }
  ]
}
```

Once you have created an ingest pipeline, you need to create an index for document ingestion. To learn more, see [Text chunking]({{site.url}}{{site.baseurl}}/search-plugins/text-chunking/).

## Cascaded text chunking processors

You can chain multiple text chunking processors together. For example, to split documents into paragraphs, apply the `delimiter` algorithm and specify the parameter as `\n\n`. To prevent a paragraph from exceeding the token limit, append another text chunking processor that uses the `fixed_token_length` algorithm. You can configure the ingest pipeline for this example as follows:

```json
PUT _ingest/pipeline/text-chunking-cascade-ingest-pipeline
{
  "description": "A text chunking pipeline with cascaded algorithms",
  "processors": [
    {
      "text_chunking": {
        "algorithm": {
          "delimiter": {
            "delimiter": "\n\n"
          }
        },
        "field_map": {
          "passage_text": "passage_chunk1"
        }
      }
    },
    {
      "text_chunking": {
        "algorithm": {
          "fixed_token_length": {
            "token_limit": 500,
            "overlap_rate": 0.2,
            "tokenizer": "standard"
          }
        },
        "field_map": {
          "passage_chunk1": "passage_chunk2"
        }
      }
    }
  ]
}
```
{% include copy-curl.html %}

## Next steps

- For a complete example, see [Text chunking]({{site.url}}{{site.baseurl}}/search-plugins/text-chunking/).
- To learn more about semantic search, see [Semantic search]({{site.url}}{{site.baseurl}}/search-plugins/semantic-search/).
- To learn more about sparse search, see [Neural sparse search]({{site.url}}{{site.baseurl}}/search-plugins/neural-sparse-search/).
- To learn more about using models in OpenSearch, see [Choosing a model]({{site.url}}{{site.baseurl}}/ml-commons-plugin/integrating-ml-models/#choosing-a-model).