## AnnotationQuery

AnnotationQuery provides a suite of composable functions to query annotations stored as a parquet file.  While the annotations will typically be generated by popular text analytic tools such as Stanford Core or Genia, the only requirement is the annotations adhere to the AQAnnotation structure.  The underlying implementation leverages Datasets and Spark(SQL).


#### CATAnnotation and AQAnnotation

While it might seem odd that there are two different Annotation classes, this was purposely done.  CATAnnotation is the archive format for the annotations and AQAnnotation is the runtime format that is used by AnnotationQuery.  The decision to use two separate classes was to insulate the AnnotationQuery implementation from the archive format providing flexibility for future optimizations.  The AQAnnotation record should never be archived (only the CATAnnotation records).  

To use AnnotationQuery, the annotations need to be in a parquet file with the following record structure.  The other field provides the option of specifying name-value pair attributes for the annotation.  For example, if the annotation and 2 attributes (color=red and size=xl), the other field would have the value color=red&size=xl.


```
case class CATAnnotation(docId: String,                 // Document Id (PII)
                         annotSet: String,              // Annotation set (such as scnlp, ge)
                         annotType: String,             // Annotation type (such as text, sentence)
                         startOffset: Long,             // Starting offset for the annotation                          
                         endOffset: Long,               // Ending offset for the annotation                          
                         annotId: Long,                 // Annotation Id 
                         other: Option[String] = None)  // Contains any attributes  (name-value pairs ampersand delimited)
```

At runtime, AnnotationQuery uses the following record structure.  A utility function is provided to transform a CATAnnotation record into a AQAnnotation record.  The name-value pairs defined in the other column (of CATAnnotation) will be converted to a Map with the name-value pairs.

```
case class AQAnnotation(docId: String,                                   // Document Id (PII)
                         annotSet: String,                               // Annotation set (such as scnlp, ge)
                         annotType: String,                              // Annotation type (such as text, sentence)
                         startOffset: Long,                              // Starting offset for the annotation
                         endOffset: Long,                                // Ending offset for the annotation
                         annotId: Long,                                  // Annotation Id
                         properties: Option[scala.collection.Map[String,String]] = None)  // Properties
```

#### Utilities

The GetAQAnnotation and GetCATAnnotation and utility classes have been developed to create an AQAnnotation from the archive format (CATAnnotation) and vise versa.  These are both defined in the com.elsevier.aq.utilities package. When creating the AQAnnotation,  the ampersand separated string of name-value pairs in the CATAnnotation other field is mapped to a Map in the AQAnnotation record.  To minimize memory consumption and increase performance, you can specify which name-value pairs to include in the Map.  For usage examples, view the GetAQAnnotation and GetCATAnnotation classes in the test package.


#### AnnotationQuery Functions

The following functions are currently provided by AnnotationQuery. Since functions return a Dataset of AQAnnotations, it is possible to nest function calls.  For more details on the implementation, view the corresponding class for each function in the com.elsevier.aq.query package.  For usage examples, view the QuerySuite class in the test package.

**FilterProperty**  -  Provide the ability to filter a property field with a specified value in a Dataset of AQAnnotations. A single value or an array of values can be used for the filter comparison.

**RegexProperty**  -  Provide the ability to filter a property field using a regex expression in a Dataset of AQAnnotations.

**FilterSet**  -  Provide the ability to filter the annotation set field in a Dataset of AQAnnotations.

**FilterType**  -  Provide the ability to filter the annotation type field in a Dataset of AQAnnotations.

**Contains**  - Provide the ability to find annotations that contain another annotation. The input is 2 Datasets of AQAnnotations. We will call them A and B. The purpose is to find those annotations in A that contain B. What that means is the start/end offset for an annotation from A must contain the start/end offset from an annotation in B. We of course have to also match on the document id. We ultimately return the container annotations (A) that meet this criteria. We also deduplicate the A annotations as there could be many annotations from B that could be contained by an annotation in A but it only makes sense to return the unique container annotations. There is also the option of negating the query (think Not Contains) so that we return only A where it does not contain B.

**ContainedIn**  -  Provide the ability to find annotations that are contained by another annotation. The input is 2 Datasets of AQAnnotations. We will call them A and B. The purpose is to find those annotations in A that are contained in B. What that means is the start/end offset for an annotation from A must be contained by the start/end offset from an annotation in B. We of course have to also match on the document id. We ultimately return the contained annotations (A) that meet this criteria. There is also the option of negating the query (think Not Contains) so that we return only A where it is not contained in B.

**Before**  -  Provide the ability to find annotations that are before another annotation. The input is 2 Datasets of AQAnnotations. We will call them A and B. The purpose is to find those annotations in A that are before B. What that means is the end offset for an annotation from A must be before the start offset from an annotation in B. We of course have to also match on the document id. We ultimately return the A annotations that meet this criteria. A distance operator can also be optionally specified. This would require an A annotation (endOffset) to occur n characters (or less) before the B annotation (startOffset). There is also the option of negating the query (think Not Before) so that we return only A where it is not before B.

**After**  -  Provide the ability to find annotations that are after another annotation. The input is 2 Datasets of AQAnnotations. We will call them A and B. The purpose is to find those annotations in A that are after B. What that means is the start offset for an annotation from A must be after the end offset from an annotation in B. We of course have to also match on the document id. We ultimately return the A annotations that meet this criteria. A distance operator can also be optionally specified. This would require an A annotation (startOffset) to occur n characters (or less) after the B annotation (endOffset). There is also the option of negating the query (think Not After) so that we return only A where it is not after B.

**Between**  -  Provide the ability to find annotations that are before one annotation and after another. The input is 3 Datasets of AQAnnotations. We will call them A, B and C. The purpose is to find those annotations in A that are before B and after C. What that means is the end offset for an annotation from A must be before the start offset from an annotation in B and the start offset for A be after the end offset from C. We of course have to also match on the document id. We ultimately return the A annotations that meet this criteria. A distance operator can also be optionally specified. This would require an A annotation (endOffset) to occur n characters (or less) before the B annotation (startOffset) and would require the A annotation (startOffset) to occur n characters (or less) after the C annotation (endOffset) . There is also the option of negating the query (think Not Between) so that we return only A where it is not before B nor after C.

**Sequence**  -  Provide the ability to find annotations that are before another annotation. The input is 2 Datasets of AQAnnotations. We will call them A and B. The purpose is to find those annotations in A that are before B. What that means is the end offset for an annotation from A must be before the start offset from an annotation in B. We of course have to also match on the document id. We ultimately return the annotations that meet this criteria. Unlike the Before function, we adjust the returned annotation a bit. For example, we set the annotType to "seq" and we use the A startOffset and the B endOffset. A distance operator can also be optionally specified. This would require an A annotation (endOffset) to occur n characters (or less) before the B annotation (startOffset).

**Or**  -  Provide the ability to combine (union) annotations. The input is 2 Datasets of AQAnnotations. The output is the union of these annotations.

**And**  -  Provide the ability to find annotations that are in the same document. The input is 2 Datasets of AQAnnotations. We will call them A and B. The purpose is to find those annotations in A and B that are in the same document.

**MatchProperty**  -  Provide the ability to find annotations (looking at their property) that are in the same document. The input is 2 Datasets of AQAnnotations. We will call them A and B. The purpose is to find those annotations in A that are in the same document as B and also match values on the specified property.

**Preceding**  -  Return the preceding sibling annotations for every annotation in the anchor Dataset[AQAnnotations]. The preceding sibling annotations can optionally be required to be contained in a container Dataset[AQAnnotations]. The return type of this function is different from other functions. Instead of returning a Dataset[AQAnnotation] this function returns a Dataset[(AQAnnotation,Array[AQAnnotation])].

**Following**  -  Return the following sibling annotations for every annotation in the anchor Dataset[AQAnnotations]. The following sibling annotations can optionally be required to be contained in a container Dataset[AQAnnotations]. The return type of this function is different from other functions. Instead of returning a Dataset[AQAnnotation] this function returns a Dataset[(AQAnnotation,Array[AQAnnotation])].


#### Concordancers

The following functions have proven useful when looking at AQAnnotations.  When displaying an annotation, the starting text for the annotation will begin with a green ">" and end with a green "<". If you use the XMLConcordancer that outputs the original XML (from the OM annotations), the XML tags will be in orange. The XML may not be well-formed). When generating annotations, you sometimes may want to exclude some text.  In AQAnnotations, this is done with the excludes property.  When an annotation is encountered that has an excludes property, the text excluded will be highlighted in red.  For more details on the implementation, view the corresponding class for each function in the com.elsevier.aq.concordancers package.  For usage examples, view the ConcordancerSuite class in the test package.

**Concordancer**  -  Output the string of text identified by the AQAnnotation and highlight in 'red' the text that was ignored (excluded).

**XMLConcordancer**  -  Output the string of text identified by the AQAnnotation and highlight in 'red' the text that was ignored (excluded). Also add the XML tags (in 'orange') that would have occurred in this string. Note, there are no guarantees that the XML will be well-formed.

**OrigPosLemConcordancer**  -  Output the string of text identified by the AQAnnotation (typically a sentence annotation). Below the sentence (in successive rows) output the original terms, parts of speech, and lemma terms for the text identified by the AQAnnotation.


#### Citing

If you need to cite AnnotationQuery in your work, please use the following DOI:

[![DOI](https://zenodo.org/badge/99150085.svg)](https://zenodo.org/badge/latestdoi/99150085)

McBeath, Darin (2017). AnnotationQuery [Computer Software];https://github.com/elsevierlabs-os/AnnotationQuery

