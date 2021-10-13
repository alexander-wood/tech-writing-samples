# Notes on the docs
## HOWTO guide: Regions
### Methodology
In this document, I assume that the reader already has `flyctl` installed and has a Fly.io application. I walk them through the required commands to run an application in multiple regions, as well as configure a PostgreSQL database and Redis store. I try to convey the information in a friendly, approachable manner and maintain a balance between length and utility.

I chose the scope because I believe nothing irritates a developer quite like "friendly" docs which omit important information or fail to warn you about inappropriate use cases. Therefore, I included a warning about use cases where multi-region Postgres would not be a good choice and linked to the explanation in the existing documentation. 

### Testing Effectiveness
I would evaluate the documents in terms of purpose, quality, and usability. 

A walkthrough should allow the reader to perform a task without company assistance. To measure how fit for purpose it is, I would compare the number of support requests before and after publication. Another indicator is the amount of time the team spends answering questions about specific topics. 

There are many ways to measure a document's quality - accuracy, completeness, user-friendliness, and so on. People's opinions may differ about which are more important. The user is the person whose opinion on the document's quality matters. Therefore, to measure quality, I would ask the users what their expectations of quality are, identify the top 1-3 aspects which users are looking for, and create metrics around them. To get this information, there could be usability studies or conversations with the community. 

The way to measure the doc's usability is a usability study. Ask people to follow the walkthrough while I observe, and then take in feedback after they're finished.

## Reference doc: Static asset caching
### Methodology
In this doc, I assume that the reader is someone seeking "just the facts". As a result, I try to convey the information in as brief and straightforward a manner as possible. I chose to follow the general pattern as elsewhere in the configuration reference doc - consistency is important.

I noticed in the community thread on the feature that there were a few feature requests which were under discussion (`cache-control` headers, HTTP proxy, 304 if the etag matches). They still seemed like open questions, so I didn't include them.

### Testing Effectiveness
A reference document should provide comprehensive, pertinent details for consultation about a subject. To measure how effective it is, I would compare the number of support requests before and after publication, as well as posts which are easily answered with a link to the document.

In terms of quality, I think completeness and accuracy are more important in reference documentation than elsewhere. To measure this, I would ask a subject matter expert for review.

To evaluate the document's usability, I would devise some tasks the document could be used for and ask them to use the documents to perform them while I observe. (IE, perhaps there might be a sample application which is missing the `statics` sections, and the participant has to use the reference docs to 'fill in the blanks'.)