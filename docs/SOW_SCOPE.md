# Engagement scope of work

Scope text of the executed engagement statement of work (Exhibit A of the
Promotional Services Agreement dated April 27, 2026). Commercial figures and
billing terms are intentionally omitted from this copy, per this repository's
no-commercial-terms rule. Work item numbering matches
[the status page](SOW_COMPLIANCE.md).

## 1. Scope of Work

### 1.1 KV Cache System and Benchmarking

During inference, models generate Key/Value tensors that represent prior
tokens. This is a new space, and the requirements are putting new constraints
on Valkey. Specifically, while the majority of use cases for Valkey work with
smaller objects of under 1MB, KV Cache workflows frequently demand objects of
10MB. Furthermore, there are specific open source projects like LMCache that
warrant specialized client side optimizations to make Valkey competitive with
alternative storage options available. While there is an integration for
Valkey with LMCache, it is not keeping up with alternative storage options.

**1.1.1 Optimize the LMCache Valkey Connector.** Momento shall establish a
baseline for Valkey connector for LMCache by 4/30/2026. Subsequently, Momento
shall deliver at least a 2x improvement in large-object (>1MB) throughput
compared to the established baseline, through retrieval optimizations (e.g.,
parallel fetch and prefetch strategies, optimal i/o thread configs, etc).
Performance will be measured relative to the existing LMCache Valkey connector
under equivalent workloads.

**1.1.2 Contribute optimized LMCache Connector.** Momento shall publish the
optimized connector, recommended configurations, and benchmark results to
either the LMCache repository or on a publicly accessible repository with a
BSD, MIT, or Apache 2.0 open source license. LMCache is currently licensed
with Apache 2.0.

**1.1.3 Benchmark Tooling for KV Cache.** Momento shall deliver reproducible
benchmarks with scripts, workloads, and representative scenarios (including
document corpus of at least 30 legal / medical docs) to measure throughput,
latency, and resource efficiency of LMCache connectors for Valkey.

**1.1.4 Investigate RDMA with Valkey for KV Cache.** Momento shall investigate
and deliver a technical report to Amazon on the effectiveness of RDMA for KV
Caches backed by Valkey.

### 1.2 AI Observability, Search, and Benchmarking

**1.2.1 Develop 1 demo integrating Valkey Search into an Inference Workflow.**
Momento shall develop this demo and contribute it to an open source
repository, like Valkey for AI, with a BSD, MIT, or Apache 2.0 license. This
demo is intended to help the audience understand Valkey's role in inference.

**1.2.2 Develop a benchmarking harness for Valkey Search.** Performance for
Valkey vector search is crucial for adoption. Valkey-bench and Valkey-lab are
currently focused only on latency / throughput of standard key-value and data
structures. Valkey search requires a harness that can assess its performance
on standard datasets held at a specific recall.

Momento shall deliver a harness that: a) loads a data set into Valkey, b)
measures indexing time, and c) runs queries with a single client to assess
throughput and latency for specific recall. The ef_search parameters and
recall targets would be configurable by the end user. The harness will report
(on a terminal) the recall, latency, and throughput.

**1.2.3 Produce a technical report outlining performance of Valkey Search.**
Momento shall deliver a report to Amazon after executing the Valkey Search
benchmarking harness developed in 1.2.2 against up to 3 versions of vector
search. The final report will be made available to the Valkey community and
will be used to produce one of the blogs in 1.3.1.

### 1.3 Content, Conference, and Community Building. #ValkeyForAI

**1.3.1 Publish 6 Technical Blogs Highlighting AI Specific Benchmarks for
Valkey.** Momento shall produce 6 blogs over the course of this SOW (at least
1 in May, 1 in June, and 1 in July) to keep a consistent cadence. These blogs
will cover search benchmarking results, KV Cache improvements, and demos to
encourage Valkey's adoption in AI workflows. Momento shall publish the blogs
on the Valkey for AI site, Valkey.io, Momento's site, or on a reputable site
like InfoQ; provided that Amazon must provide prior written approval of the
specific publication venue before each blog is published.

**1.3.2 Develop 2 technical presentations on KV Cache and Search Performance
in Valkey.** Momento shall produce 2 technical presentations that cover
Valkey's performance on KV Cache and Search. These presentations must be
reviewed and approved by an L7 (or above) Amazon stakeholder for quality and
rigor before use. These are the presentations Momento is expected to deliver
at the conferences and events described in 1.3.3.

**1.3.3 Present Valkey Search and Valkey KV Cache Capabilities at recognized
technical conferences.** Momento shall present Valkey KV Cache and/or Vector
search at QConAI Boston (June '26). Momento shall also curate an AI
presentation block for an upcoming Unlocked (3-5 AI sessions with Valkey
components). Not all sessions have to be presented by Momento for the
Unlocked, as the intent is to produce and curate the content across the
community.

**1.3.4 Publish 4 podcasts on Valkey for KV Cache and Search.** Momento shall
produce 4 podcasts with industry experts on KV Cache and Search to promote AI
use cases for Valkey. The podcast will be available on the Valkey YouTube
channel via direct publication or in the form of a playlist.

## 2. Payment Terms

Omitted from this copy (commercial terms). Term: 4 months.

## 3. Reporting and Communication

- Monthly progress updates from Momento
- Blogs and podcasts are distributed evenly through the work period
- Amazon will provide a single point of contact for feedback and approvals
