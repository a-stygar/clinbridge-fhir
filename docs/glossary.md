# ClinBridge FHIR Glossary

## FHIR
Fast Healthcare Interoperability Resources - це HL7 стандарт електронного обміну
медичною інформацією між незалежними системами. Основними блоками цього стандарту є
ресурси - наприклад Patient, Observation, ServiceRequest. FHIR визначає інформаційну
модель, типи даних, references, правила взаємодії, профілі та кілька способів обміну
інформацією.

## Resource
Основний будівельний блок FHIR який формує інформаційну модель стандарта.
Наприклад: `Patient`, `Obsvation`, `ServiceRequest`.

element - поле ресурса
datatype - тип елемента
Reference - зв'язок між ресурсами
profile - спосіб обмежити чи уточнити ресурс на рівні країни, організації
Implementation Guide
CapabilityStatement
FHIR server - конкретна реалізація FHIR, у нашому випадку FHIR HAPI
integration engine
