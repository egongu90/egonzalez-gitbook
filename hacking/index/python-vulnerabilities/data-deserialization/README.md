# Data Deserialization

## What is data serialization?

Data serialization is the process to take some python object and transform into structured data that can be consumed by different backend technologies.

Deserialization is the opposite way of serialization, is the process of taking data structured and transform into an object to be shared through different phases of the code.

Is commonly used by backend services to transport and manage data through different classes, libraries,  update information about the object attributes and finally transform into structured data to be sent into a database, storage service or represented in a REST API.

Some of the most common data structure formats used are:

* json
* yaml
* xml
* pickle
* csv

## Vulnerabilities

If the data format and the development methods used are able to evaluate python code from the data, an attacker may be able to inject it's python object and execute it in the backend service. Possibly allowing remote shells or information disclosure.







