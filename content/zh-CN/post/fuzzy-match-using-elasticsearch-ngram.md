---
title: 使用 Elasticsearch 的 NGram 分词器处理模糊匹配
date: 2020-01-28
tags:
  - elasticsearch
  - ngram
  - fuzzy match
---

接到一个任务：用 Elasticsearch 实现搜索银行支行名称的功能。大概就是用户输入一截支行名称或拼音首字母，返回相应的支行名称。比如，用户输入"工行"或者"gh"，我需要返回"工行XXX分行"类似这样的结果。

我心里嘀咕着：数据库不是支持通配符查询吗？为什么不直接用数据库查询？

说归说，但是任务还是要完成的。之前有在网上看过一篇文章，主要就是说用 Elasticsearch 处理通配符查询不太适合，然后我在评论中看到作者推荐了一个分词器 [NGram](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/_ngrams_for_partial_matching.html#_ngrams_for_partial_matching)。

这个分词器可以让通配符查询和普通的查询一样迅速，因为该分词器在数据索引阶段就把所有工作做完了：
```
An n-gram can be best thought of as a moving window on a word. The n stands for a length. If we were to n-gram the word quick, the results would depend on the length we have chosen:
Length 1 (unigram): [ q, u, i, c, k ]
Length 2 (bigram): [ qu, ui, ic, ck ]
Length 3 (trigram): [ qui, uic, ick ]
Length 4 (four-gram): [ quic, uick ]
Length 5 (five-gram): [ quick ]
```
若要使用 NGram 分词器作为某个字段的分词器，可在索引创建时指定，也可以更新映射关系，以下展示如何在索引创建时指定 NGram 分词器。
```
{
  "settings": {
    "analysis": {
      "analyzer": {
        "ngram_analyzer": {
          "tokenizer": "ngram_tokenizer"
        }
      },
      "tokenizer": {
        "ngram_tokenizer": {
          "type": "ngram",
          "min_gram": 1,
          "max_gram": 30,
          "token_chars": [
            "letter",
            "digit"
          ]
        }
      }
    }
  },
  "mappings": {
        "_default_": {
            "properties": {
                "Name": {
                    "type": "string",
                    "analyzer": "ngram_analyzer"
                }
            }
        }
    }
}
```
当某个字段的 analyzer 被指定为 ngram_analyzer，这个字段的查询就都会变成通配符查询，无论是 term 还是 match。

比如，
```
POST /index/type
{
    "query": {
        "term": {"Name": "工商"}
    }
}
```
会得到"中国工商银行XXX分行"。

比如，
```
POST /index/type
{
    "query": {
        "match": {"Name": "工商"}
    }
}
```
会得到"中国工商银行XXX分行"、"工行XXX分行"、"中国招商银行XXX分行"。

match 查询会对关键词进行分词，而 Lucene 的默认中文分词就是把每个中文字拆开，这样会变成对"工"、"商"两个字做通配符查询。