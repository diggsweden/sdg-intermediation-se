![Logo](/images/digg.png)

# SDG Bevisdelning, inom det tekniska systemet för bevisutbyte
Här finns beskrivningen av API:et intermediationSE vilket är det API som svenska bevisproducenter implementerar, för att via Diggs förhandsgranskningstjänst, låta en användare dela bevis via det tekniska systemet för bevisutbyte (OOTS).

## Bevisdelning och förhandsgranskning, översiktligt flöde
```mermaid
flowchart LR
    FGT(Förhandsgranskningstjänsten)-->AT(SDG Auktorisationstjänst)
    AT-->LT(Legitimeringstjänst)
    FGT-->BT(Bevistjänst)
```

### Flödesbeskrivning
1. 

## Bevisdelning och förhandsgranskning, detaljerat flöde
```mermaid
sequenceDiagram
    participant USR as User
    participant PDFE as ProcedureDemoFE
    participant PSFE as PreviewSpaceFE
    participant PSBE as PreviewSpaceBE
    participant IMSE as IntermediationSE
    participant RED as Redis
    box Auktorisation
    participant AS as AuthorizationServer
    participant ATS as AuthenticationService
    end
    participant EP as EvidenceProvider

    USR ->> PDFE: 1. Start procedure
    PDFE ->> PSFE: 2. Redirect to preview
    PSFE ->> PSBE: 3. GET /authorize/login

    PSBE ->> AS: 4. AuthenticationRequest, with redirect uri, to AuthorizationServer
    Note over AS,ATS: 5. User authentication
    AS ->> PSBE: 6. Response from AuthorizationServer to PSBE including authentication code
    PSBE ->> AS: 7. IdTokenRequest(code)
    AS ->> PSBE: 8. TokenResponse(idtoken, accesstoken, refreshtoken)
    PSBE ->> RED: 9. Store accesstoken for reuse
    PSBE ->> PSFE: 10. Redirect from PSBE to PSFE /login
    PSFE ->> PSBE: 11. GET /authorize/authorizationRequest
    PSBE ->> AS: 12. AuthorizationRequest, with redirect uri, to AuthorizationServer
    Note over AS,ATS: 13. Evidence provider authorization
    AS ->> PSBE: 14. Redirect from AuthorizationServer to PSBE including authorization code
    PSBE ->> AS: 15. AccessTokenRequest(code)
    AS ->> PSBE: 16. TokenResponse(accesstoken, refreshtoken)
    PSBE ->> RED: 17. Store accesstoken for reuse
    PSBE ->> PSFE: 18. Redirect from PSBE to PSFE
    PSFE ->> PSBE: 19. Post /documents/available to fetch evidences
    PSBE ->> AS: 20. Get user info (accesstoken)
    AS ->> PSBE: 21. User info response
    PSBE ->> PSBE: 22. Match evidence request against user information
    PSBE ->> IMSE: 23. POST /evidence-files (accesstoken)
    IMSE ->> EP: 24. GET evidence files from provider (accesstoken)
    EP ->> EP: 25. Validate accesstoken
    EP -->> IMSE: 26. Return documents
    IMSE -->> PSBE: 27. Return documents
    PSBE -->> PSFE: 28. Show documents
    PSFE -->> PDFE: 29. Share documents
```

### Flödesbeskrivning
1. Användaren besöker förfarandet och förbereder en bevisbegäran
2. Användaren initierar autentisering
3. Användaren väljer att logga in och initierar legitimeringsförfrågan
4. PreviewSpaceBE bygger ihop och skickar ett authentication request till AuthorizationServer, samtidigt blir användaren omdirigerad till id-tjänsten
5. Användaren legitimerar sig i id-tjänsten
6. AuthorizationServer svarar förbestämd callback endpoint med en authentication code
7. PreviewSpaceBE bygger ihop och skickar ett idtoken request till AuthorizationServer
8. Authorization svarar med idtoken, accesstoken, refreshtoken.
9. PreviewSpaceBE sparar ner accesstoken i redis.
10. Användaren blir omdirigerad tillbaka till PSFE
11. Användaren initierar auktorisation
12. PreviewSpaceBE bygger ihop och skickar ett authorization request till AuthorizationServer
13. Auktorisering av klient
14. AuthorizationServer svarar förbestämd callback endpoint med en authorization code
15. PreviewSpaceBE bygger ihop och skickar ett accesstoken request
16. AuthorizationServer svarar med accesstoken och refreshtoken
17. PreviewSpaceBE sparar ner accesstoken i redis
18. Användaren blir omdirigerad tillbaka till PSFE
19. Bevishämtning initieras
20. PreviewSpaceBE skickar en förfrågan till AuthorizationServer för att hämta användarinformation
21. AuthorizationServer svarar med användarinformation
22. PreviewSpaceBE matchar information från evidence request mot användarinformationen.
23. PreviewSpaceBE skickar förfrågan, som innehåller accesstoken, för bevishämtning till IntermediationSE
24. IntermediationSE skickar förfrågan, som innehåller accesstoken, till bevisproducent
25. Bevisproducent validerar accesstoken
26. Bevisproducent returnerar IntermediationSE med bevis
27. IntermediationSE returnerar bevis till PreviewSpaceBE
28. PreviewSpaceBE returnerar bevis till PreviewSpaceFE som visar upp hämtade bevis för användaren.
29. Användare väljer att dela bevis med förfarande

