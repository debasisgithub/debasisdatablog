---
title: "Azure Active Directory"
datePublished: Fri Jan 17 2025 21:45:32 GMT+0000 (Coordinated Universal Time)
cuid: cm61agepf000209l5gd70bakl
slug: azure-active-directory
tags: azure, databricks

---

Azure Active Directory or AAD is the Microsoft cloud based access identity management service. It sits at tenant level hierarchy, meaning there will be only one AAD per tenant and all subscriptions under the tenant use the same AAD .

AAD is a directory of objects. Human users, applications, Service Principals are also objects in the directory.

There are two broad category of user types in Azure AD:

* B2B (Business-to-Business) users
    
    This users are not tied to the business domain but are added to the AAD, they might be employee of contact company or vendor company. Below are the list of B2B users.
    
    **Vendors**
    
    **Contractors**
    
    **Partners**
    
    **Non-employee/direct addition to the AAD**
    
    **Guest roles**
    
* B2C (Business-to-Customer) users
    
    **Authenticated via 3rd party identity provider such as google, facebook**
    
    **Not added to AAD**
    
    **Limited access, cannot use Azure Databricks workspace.**