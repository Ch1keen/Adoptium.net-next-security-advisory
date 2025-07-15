# Open Redirect on Adoptium.net

Affected Version: Before commit 1f9753c25fccfc4ee37b63476cce31e46220ac77 (Committed at 2025.07.03)

Fixed Version: Since commit 1f9753c25fccfc4ee37b63476cce31e46220ac77. [Browse the Commit](https://github.com/adoptium/adoptium.net/commit/1f9753c25fccfc4ee37b63476cce31e46220ac77)

## How to reproduce

Pass an external URL to the link​ query parameter.

For example, if a user clicks the link below

```
hxxps://adoptium.net/download?link=hxxps%3A%2F%2Fexample%2Ecom&vendor=Adoptium
```

they will be redirected to `hxxps://example.com`.

## Impact

An attacker could craft a phishing URL that begins with the trusted Adoptium domain and then redirects victims to a malicious site.

This vulnerability may be abused that scammers can lure people to steal information and credentials of users who clicked the link.

And it can also be used to bypass protection mechanism of certain system that relies to domain allow lists.

## Root Cause

Most of Open Redirect Vulnerability is due to improper validation of GET parameter. The `link` query parameter of `/download` page accepts any website, and `'httpEquiv="refresh"'` is triggered.

```jsx
...
    return (
        <div>
            <PageHeader
                title="Thank you for your download!"
                subtitle="Download Success"
                description={
                    <>
                        {link && (
                            <>
                                <meta httpEquiv="refresh" content={`0; url=${link}`} />
...
```

According to line 14 to line 16, there is no validation on the parameter. Only if `link` query parameter is empty, the site renders `/temurin/releases`.

```jsx
export default function DownloadPageClient() {
    const searchParams = useSearchParams()
    const link = searchParams.get("link") || "";
    const vendor = searchParams.get("vendor") || "Adoptium";

    if (!link) {
        redirect("/temurin/releases");
    }
...
```

Restricting domain by maintaining allow lists of would prevent scammers to abuse `/download` page.

## The Patch

First, a patch suggested the `link` parameter to be validated by checking whether the `link` parameter starts with `https://github.com/adoptium/temurin`.

```diff
@@ -11,7 +11,7 @@ export default function DownloadPageClient() {
<   if (!link) {
---
>   if (!link || !link.startsWith("https://github.com/adoptium/temurin")) {
```

Next, another patch was commited to validate link based on an allow list.

```diff
@@ -11,7 +11,24 @@ export default function DownloadPageClient() {
<   if (!link || !link.startsWith("https://github.com/adoptium/temurin")) {
---
>   if (!link) {
>       redirect("/temurin/releases");
>   }
>
>   // Validate allowed download link origins for security
>   const allowedOrigins = [
>       "https://github.com/adoptium/temurin",
>       "https://cdn.azul.com/zulu/",
>       "https://aka.ms/download-jdk/",
>       "https://github.com/ibmruntimes/",
>       "https://github.com/dragonwell-project/",
>       "https://developers.redhat.com/"
>   ];
>
>   // Use Array.some for efficient matching and avoid prototype pollution
>   const isValidLink = allowedOrigins.some(origin => link.startsWith(origin));
>   if (!isValidLink) {
>       console.error("Invalid download link:", link);
```

The first patch: https://github.com/adoptium/adoptium.net/commit/1f9753c25fccfc4ee37b63476cce31e46220ac77

The next patch: https://github.com/adoptium/adoptium.net/commit/5048e0d67c3cc8e1f2038a72537918bc5c20248c

## Disclosure Timeline

* 2025.07.03 12:30 pm KST: Vulnerability reported to maintainers of the site.
* 2025.07.03: The vulnerability was acknowledged.
* 2025.07.03: Github assigned GHSA-wfq4-6v32-jrhq.
* 2025.07.03: The maintainer fixed the vulnerability.
* 2025.07.15: Public Disclosure.
