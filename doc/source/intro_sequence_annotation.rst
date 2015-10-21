---------------------------------------
Annotation Design - RNA Considerations
---------------------------------------

Read data derived from RNA samples can differ from genomic read data due to the presence of non-genomic sequences.  An example would be a read that spans a splice junction.  It describes a contiguous sequence of reads, but a dis-continuous genomic region due to the missing intron.  Feature level read assignment is further complicated by the existence of multiple splice isoforms.  A read that can be definitely assigned to a particular feature (an exon in this case) may still not be definitely assigned to a particular transcript if multiple transcript share that exon.  The annotation API needs to be able to report assignment at the feature level as well as aggregate assignment at the transcript or even the whole gene level if assignment is not more specific than that.

Splicing (other post-transcriptional modifications?) can occur with degrees of complexity.  A ‘typical’ splice will result in a mature transcript with exon in positional (numerical) order in a head-to-tail orientation.  Back splicing (tail-to-head) can result in transcripts with the exon order reversed (1-3-2-4 instead of 1-2-3-4) and even circular RNA.  The exon order in a transcript as well as the orientation of the splice should be discoverable via the API.  In a more general case, the API should allow child features to have an ordered relationship.

The annotation API needs to also be flexible enough to handle multiple references in the same gene or transcript.  This is needed to cover the cases of fusion genes or inter-chromosomal translocations.
