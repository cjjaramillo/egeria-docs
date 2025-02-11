<!-- SPDX-License-Identifier: CC-BY-4.0 -->
<!-- Copyright Contributors to the ODPi Egeria project 2020. -->

# Defining new types

Egeria supports a [wide variety of open metadata types](/types) that cover a broad range of metadata.  These types can be extended through the normal contribution process of the Egeria project.   It is also possible to create your own type definitions that can be exchanged across the [open metadata repository cohort](/concepts/cohort-member) and may also be maintained through Egeria's APIs using [extended properties](/parameters/overview/#extended-properties).

Type definitions need to be governed because they must be consistent across a cohort - during the whole time that one or more instances of the type are stored in the repositories.

## Dynamic definitions of types not recommended

Egeria does have an API for dynamically defining types, but it is recommended that you only use it went experimenting with types and instances.  Instead, you should manage your types in an [open metadata archive](/concepts/open-metadata-archive) that is loaded in the [metadata access servers](/concepts/metadata-access-servers) during [each server start up](/guides/admin/servers/configuring-a-metadata-access-store/#configure-metadata-to-load-on-startup).

## Creating your own type archive

A type archive is created using a Java program that uses the [repository services archive utilities](https://github.com/odpi/egeria/tree/main/open-metadata-implementation/repository-services/repository-services-archive-utilities) to add your new type definitions to an open metadata archive file.


Begin by setting up your build file - this is an example using Maven.  The Java class where the `main()` method is located is called `cocopharma.openmetadata.types.CocoTypesArchiveUtility` in this example.

??? example "Build file"
    ```
    
    <?xml version="1.0" encoding="UTF-8"?>
    
    <!-- SPDX-License-Identifier: Apache-2.0 -->
    <!-- Copyright Contributors to the ODPi Egeria project. -->
    
    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    
        <modelVersion>4.0.0</modelVersion>
    
        <name>My Types Utility</name>
        <description>
            This package builds the open metadata type archive for my extended types.
        </description>
    
        <artifactId>my-types-utility</artifactId>
    
        <dependencies>
            <dependency>
                <groupId>org.odpi.egeria</groupId>
                <artifactId>repository-services-apis</artifactId>
            </dependency>
    
            <dependency>
                <groupId>org.odpi.egeria</groupId>
                <artifactId>repository-services-archive-utilities</artifactId>
            </dependency>
    
            <dependency>
                <groupId>org.odpi.egeria</groupId>
                <artifactId>open-metadata-types</artifactId>
            </dependency>
        
        </dependencies>
    
        <build>
            <plugins>
                <plugin>
                    <groupId>org.apache.maven.plugins</groupId>
                    <artifactId>maven-assembly-plugin</artifactId>
                    <executions>
                        <execution>
                            <id>assemble-all</id>
                            <phase>package</phase>
                            <goals>
                                <goal>single</goal>
                            </goals>
                            <configuration>
                                <descriptorRefs>
                                    <descriptorRef>jar-with-dependencies</descriptorRef>
                                </descriptorRefs>
                                <archive>
                                    <manifest>
                                        <mainClass>org.odpi.openmetadata.samples.archiveutilities.types.CocoTypesArchiveWriter</mainClass>
                                    </manifest>
                                </archive>
                            </configuration>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </project>
    ```

The next example shows the code to build four new types:

* *BiopsyScope* - An EnumDef that holds an enumeration.
* *BiopsyReport* - An EntityDef describing a new type of entity.
* *BiopsySupportingEvidence* - A RelationshipDef describing a type of relationship that links the biopsy report to a Referenceable element that describes the subject of the report.
* *ReviewedByClinicalTrials* - A ClassificationDef that describes a new classification that indicated an element has been reviewed by the clinical trials team.  It captures details about the reviewer and their findings.

There are three main components.  A builder that constructs the archive in memory; a helper that constructs each element for the archive and a writer that first calls the builder to extract the content and writes it to a file in JSON.

??? example "Java program to build new types"
    ```java
    /* SPDX-License-Identifier: Apache-2.0 */
    /* Copyright Contributors to the ODPi Egeria project. */
    package org.odpi.openmetadata.samples.archiveutilities.types;
    
    import org.odpi.openmetadata.opentypes.OpenMetadataTypesArchive;
    import org.odpi.openmetadata.repositoryservices.archiveutilities.OMRSArchiveWriter;
    import org.odpi.openmetadata.repositoryservices.archiveutilities.OMRSArchiveBuilder;
    import org.odpi.openmetadata.repositoryservices.archiveutilities.OMRSArchiveHelper;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.archivestore.properties.OpenMetadataArchive;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.archivestore.properties.OpenMetadataArchiveType;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.ClassificationDef;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.ClassificationPropagationRule;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.EntityDef;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.EnumDef;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.EnumElementDef;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.RelationshipDef;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.RelationshipEndCardinality;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.RelationshipEndDef;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.TypeDefAttribute;
    import org.odpi.openmetadata.repositoryservices.connectors.stores.metadatacollectionstore.properties.typedefs.TypeDefLink;
    
    import java.util.ArrayList;
    import java.util.Date;
    import java.util.List;
    
    
    /**
     * CocoTypesArchiveWriter creates a physical open metadata archive file containing
     * the additional specialized types needed by Coco Pharmaceuticals.
     */
    public class CocoTypesArchiveWriter extends OMRSArchiveWriter
    {
        private static final String cocoTypesArchiveFileName = "CocoTypesArchive.json";
    
        /*
         * This is the header information for the archive.
         */
        private static final String                  archiveGUID        = "50874908-01f1-47e2-83ea-e571109a946e";
        private static final String                  archiveName        = "CocoTypes";
        private static final String                  archiveLicense     = "Apache 2.0";
        private static final String                  archiveDescription = "Specialized types for Coco Pharmaceuticals.";
        private static final OpenMetadataArchiveType archiveType        = OpenMetadataArchiveType.CONTENT_PACK;
        private static final String                  originatorName     = "Egeria Project";
        private static final Date                    creationDate       = new Date(1639984840038L);
    
        /*
         * Specific values for initializing TypeDefs
         */
        private static final long   versionNumber = 1L;
        private static final String versionName   = "1.0";
    
    
        private final OMRSArchiveBuilder archiveBuilder;
        private final OMRSArchiveHelper  archiveHelper;
    
        /**
         * Default constructor initializes the archive.
         */
        public CocoTypesArchiveWriter()
        {
            List<OpenMetadataArchive> dependentOpenMetadataArchives = new ArrayList<>();
    
            /*
             * This value allows the Coco Types to be based on the existing open metadata types
             */
            dependentOpenMetadataArchives.add(new OpenMetadataTypesArchive().getOpenMetadataArchive());
    
            this.archiveBuilder = new OMRSArchiveBuilder(archiveGUID,
                                                         archiveName,
                                                         archiveDescription,
                                                         archiveType,
                                                         originatorName,
                                                         archiveLicense,
                                                         creationDate,
                                                         dependentOpenMetadataArchives);
    
            this.archiveHelper = new OMRSArchiveHelper(archiveBuilder,
                                                       archiveGUID,
                                                       originatorName,
                                                       creationDate,
                                                       versionNumber,
                                                       versionName);
        }
    
    
        /**
         * Create an enum type that can be used as a type of attribute in an entity, relationship or classification.
         * It defines a set of values that the attribute can be set up with along with a default value.
         *
         * @return enum attribute type definition (EnumDef)
         */
        private EnumDef getBiopsyScopeEnum()
        {
            final String guid            = "fdb05618-d0fe-4725-946f-138ba74f6f43";
            final String name            = "BiopsyScope";
            final String description     = "Defines scope of the tissue removal for a biopsy.";
            final String descriptionGUID = null;
    
            EnumDef enumDef = archiveHelper.getEmptyEnumDef(guid, name, description, descriptionGUID);
    
            ArrayList<EnumElementDef> elementDefs = new ArrayList<>();
            EnumElementDef            elementDef;
    
            final int    element1Ordinal         = 0;
            final String element1Value           = "Unclassified";
            final String element1Description     = "There is no information on the scope of the biopsy.";
            final String element1DescriptionGUID = null;
    
            elementDef = archiveHelper.getEnumElementDef(element1Ordinal,
                                                         element1Value,
                                                         element1Description,
                                                         element1DescriptionGUID);
            elementDefs.add(elementDef);
            enumDef.setDefaultValue(elementDef);
    
            final int    element2Ordinal         = 1;
            final String element2Value           = "Excisional";
            final String element2Description     = "The biopsy removed all of the suspicious tissue.";
            final String element2DescriptionGUID = null;
    
            elementDef = archiveHelper.getEnumElementDef(element2Ordinal,
                                                         element2Value,
                                                         element2Description,
                                                         element2DescriptionGUID);
            elementDefs.add(elementDef);
    
            final int    element3Ordinal         = 2;
            final String element3Value           = "Incisional";
            final String element3Description     = "The biopsy took a sample of the tissue under examination.";
            final String element3DescriptionGUID = null;
    
            elementDef = archiveHelper.getEnumElementDef(element3Ordinal,
                                                         element3Value,
                                                         element3Description,
                                                         element3DescriptionGUID);
            elementDefs.add(elementDef);
    
            final int    element99Ordinal         = 99;
            final String element99Value           = "Other";
            final String element99Description     = "Another biopsy scope.";
            final String element99DescriptionGUID = null;
    
            elementDef = archiveHelper.getEnumElementDef(element99Ordinal,
                                                         element99Value,
                                                         element99Description,
                                                         element99DescriptionGUID);
            elementDefs.add(elementDef);
    
            enumDef.setElementDefs(elementDefs);
    
            return enumDef;
        }
    
    
        /**
         * Create the type definition for BiopsyReport.
         *
         * @return entity type definition (EntityDef)
         */
        private EntityDef getBiopsyReportEntity()
        {
            final String guid = "78479770-79ae-4bd8-b0ec-bf5e60c01e66";
    
            final String name            = "BiopsyReport";
            final String description     = "Results from a patient biopsy.";
            final String descriptionGUID = null;
    
            final String superTypeName = "Document";
    
            EntityDef entityDef = archiveHelper.getDefaultEntityDef(guid,
                                                                    name,
                                                                    this.archiveBuilder.getEntityDef(superTypeName),
                                                                    description,
                                                                    descriptionGUID);
    
            /*
             * Build the attributes
             */
            List<TypeDefAttribute> properties = new ArrayList<>();
            TypeDefAttribute       property;
    
            final String attribute1Name            = "biopsyScope";
            final String attribute1Description     = "Is this biopsy excisional (targeted removal) or incisional (sample taken).";
            final String attribute1DescriptionGUID = null;
            final String attribute2Name            = "biopsyTechniqueType";
            final String attribute2Description     = "How was the biopsy taken?";
            final String attribute2DescriptionGUID = null;
    
            property = archiveHelper.getEnumTypeDefAttribute("BiopsyScope",
                                                             attribute1Name,
                                                             attribute1Description,
                                                             attribute1DescriptionGUID);
            properties.add(property);
            property = archiveHelper.getStringTypeDefAttribute(attribute2Name,
                                                               attribute2Description,
                                                               attribute2DescriptionGUID);
            properties.add(property);
    
            entityDef.setPropertiesDefinition(properties);
    
            return entityDef;
        }
    
    
        /**
         * Crete a new type of relationship to link a biopsy report to other elements - assets, glossary terms, other definitions etc, that support the
         * conclusions of the biopsy report.
         *
         * @return relationship type definition (RelationshipDef)
         */
        private RelationshipDef getBiopsySupportingEvidenceRelationship()
        {
            final String guid            = "54300f97-0140-4adb-b9a9-308514694f8d";
            final String name            = "BiopsySupportingEvidence";
            final String description     = "Link between a biopsy report and other data sources.";
            final String descriptionGUID = null;
    
            final ClassificationPropagationRule classificationPropagationRule = ClassificationPropagationRule.NONE;
    
            RelationshipDef relationshipDef = archiveHelper.getBasicRelationshipDef(guid,
                                                                                    name,
                                                                                    null,
                                                                                    description,
                                                                                    descriptionGUID,
                                                                                    classificationPropagationRule);
    
            RelationshipEndDef relationshipEndDef;
    
            /*
             * Set up end 1.
             */
            final String                     end1EntityType               = "BiopsyReport";
            final String                     end1AttributeName            = "report";
            final String                     end1AttributeDescription     = "Report that the evidence is being linked to.";
            final String                     end1AttributeDescriptionGUID = null;
            final RelationshipEndCardinality end1Cardinality              = RelationshipEndCardinality.ANY_NUMBER;
    
            relationshipEndDef = archiveHelper.getRelationshipEndDef(this.archiveBuilder.getEntityDef(end1EntityType),
                                                                     end1AttributeName,
                                                                     end1AttributeDescription,
                                                                     end1AttributeDescriptionGUID,
                                                                     end1Cardinality);
            relationshipDef.setEndDef1(relationshipEndDef);
    
    
            /*
             * Set up end 2.
             */
            final String                     end2EntityType               = "Referenceable";
            final String                     end2AttributeName            = "evidence";
            final String                     end2AttributeDescription     = "Further information to support the report.";
            final String                     end2AttributeDescriptionGUID = null;
            final RelationshipEndCardinality end2Cardinality              = RelationshipEndCardinality.ANY_NUMBER;
    
            relationshipEndDef = archiveHelper.getRelationshipEndDef(this.archiveBuilder.getEntityDef(end2EntityType),
                                                                     end2AttributeName,
                                                                     end2AttributeDescription,
                                                                     end2AttributeDescriptionGUID,
                                                                     end2Cardinality);
            relationshipDef.setEndDef2(relationshipEndDef);
    
            /*
             * Build the attributes
             */
            List<TypeDefAttribute> properties = new ArrayList<>();
            TypeDefAttribute       property;
    
            final String attribute1Name            = "notes";
            final String attribute1Description     = "Information for the clinical trials team relating to the evidence.";
            final String attribute1DescriptionGUID = null;
    
            property = archiveHelper.getMapStringStringTypeDefAttribute(attribute1Name,
                                                                        attribute1Description,
                                                                        attribute1DescriptionGUID);
            properties.add(property);
    
    
            relationshipDef.setPropertiesDefinition(properties);
    
            return relationshipDef;
        }
    
    
        /**
         * Build a classification that can be attached to an asset to indicate that a member of the clinical trials has reviewed the contents
         * and made notes of their observations and conclusions.
         *
         * @return classification type definition (ClassificationDef)
         */
        private ClassificationDef getReviewedByClinicalTrialsClassification()
        {
            final String guid = "c2fa7555-f366-4869-88f3-897d6f2ec5a4";
    
            final String name            = "ReviewedByClinicalTrials";
            final String description     = "Declares that a report or data set has been assessed by the clinical trials team.";
            final String descriptionGUID = null;
    
            final List<TypeDefLink> linkedToEntities = new ArrayList<>();
    
            linkedToEntities.add(this.archiveBuilder.getEntityDef("Asset"));
    
            ClassificationDef classificationDef = archiveHelper.getClassificationDef(guid,
                                                                                     name,
                                                                                     null,
                                                                                     description,
                                                                                     descriptionGUID,
                                                                                     linkedToEntities,
                                                                                     false);
    
            /*
             * Build the attributes
             */
            List<TypeDefAttribute> properties = new ArrayList<>();
            TypeDefAttribute       property;
    
            final String attribute1Name            = "reviewer";
            final String attribute1Description     = "Person responsible for maintaining this relationship.";
            final String attribute1DescriptionGUID = null;
            final String attribute2Name            = "reviewerTypeName";
            final String attribute2Description     = "Type of element used to identify the reviewer.";
            final String attribute2DescriptionGUID = null;
            final String attribute3Name            = "reviewerPropertyName";
            final String attribute3Description     = "Name of property used to identify the reviewer.";
            final String attribute3DescriptionGUID = null;
            final String attribute4Name            = "notes";
            final String attribute4Description     = "Information for the clinical trials team relating to the review.";
            final String attribute4DescriptionGUID = null;
    
            property = archiveHelper.getStringTypeDefAttribute(attribute1Name,
                                                               attribute1Description,
                                                               attribute1DescriptionGUID);
            properties.add(property);
            property = archiveHelper.getStringTypeDefAttribute(attribute2Name,
                                                               attribute2Description,
                                                               attribute2DescriptionGUID);
            properties.add(property);
            property = archiveHelper.getStringTypeDefAttribute(attribute3Name,
                                                               attribute3Description,
                                                               attribute3DescriptionGUID);
            properties.add(property);
            property = archiveHelper.getMapStringStringTypeDefAttribute(attribute4Name,
                                                                        attribute4Description,
                                                                        attribute4DescriptionGUID);
            properties.add(property);
    
            classificationDef.setPropertiesDefinition(properties);
    
            return classificationDef;
        }
    
    
        /**
         * Returns the open metadata type archive containing all the new type definitions.
         *
         * @return populated open metadata archive object
         */
        protected OpenMetadataArchive getOpenMetadataArchive()
        {
            /*
             * Add attribute type definitions.
             */
            this.archiveBuilder.addEnumDef(getBiopsyScopeEnum());
    
            /*
             * Add the specialized types
             */
            this.archiveBuilder.addEntityDef(getBiopsyReportEntity());
            this.archiveBuilder.addRelationshipDef(getBiopsySupportingEvidenceRelationship());
            this.archiveBuilder.addClassificationDef(getReviewedByClinicalTrialsClassification());
    
    
            /*
             * The completed archive is ready to be packaged up and returned
             */
            return this.archiveBuilder.getOpenMetadataArchive();
        }
    
    
        /**
         * Generates and writes out an open metadata archive containing the new types.
         */
        public void writeOpenMetadataArchive()
        {
            try
            {
                System.out.println("Writing to file: " + cocoTypesArchiveFileName);
    
                super.writeOpenMetadataArchive(cocoTypesArchiveFileName, this.getOpenMetadataArchive());
            }
            catch (Exception error)
            {
                System.out.println("error is " + error);
            }
        }
    }
    
    ```

This example is located in the `egeria-samples.git` repository in the [coco-metadata-archives](https://github.com/odpi/egeria-samples/tree/main/sample-metadata-archives/coco-metadata-archives) module.


--8<-- "snippets/abbr.md"
