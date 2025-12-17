---
title: >
    Example
parent: Generic JSON
nav_order: 2
---

```json
{
  "schema": {
    "name": "mailwebhook.generic",
    "version": "1"
  },
  "event": {
    "id": "6ff49aa1-7050-4ad1-95d9-2711f2ca7e88",
    "project_id": "dca29061-c4a7-4687-a8dd-24d2f26548c7",
    "route_id": "2f3713bf-88cc-46c6-aaa3-ea9d6e9d20f3",
    "created_at": "2025-12-17T16:02:33Z"
  },
  "message": {
    "message_id": "dsIL2Z9xTiKRffmacoohqw@geopod-ismtpd-10",
    "message_id_type": "original",
    "subject": "Subject",
    "date": "2025-12-17T15:49:11Z",
    "from": [
      {
        "email": "notifications@sender.com",
        "name": "Sender"
      }
    ],
    "to": [
      {
        "email": "recipient.name@recipient.com"
      }
    ],
    "headers": {
      "return-path": "<bounces+47010996-326a-recipient.name=recipient.com@em4667.sender.com>",
      "delivered-to": "3@142095",
      "received": "from NDcwMTA5OTY (unknown)\tby geopod-ismtpd-10 (SG) with HTTP\tid dsIL2Z9xTiKRffmacoohqw\tWed, 17 Dec 2025 15:49:11.151 +0000 (UTC)",
      "x-original-to": "recipient.name@recipient.com",
      "x-vadesecure-originating-ip": "149.72.53.2",
      "x-vadesecure-malware": "Clean",
      "x-vadesecure-verdict": "commercial:mce",
      "x-vadesecure-status": "MCE",
      "x-vadesecure-cause": "gggruggvucftvghtrhhoucdtuddrgeefgedrtddtgdegvdeliecutefuodetggdotefrodftvfcurfhrohhfihhlvgemucfqigfugfgfpdggtfgfnhhsuhgsshgtrhhisggvpdfgpfggqdfnfggivefnqfgfffenuceurghilhhouhhtmecufedtudenucdnofetkffnkffpifculddujedmnecujfgurheptgffhfggkffuvfesrgdttdertddtvdenucfhrhhomhepsfifohhtvgguuceonhhothhifhhitggrthhiohhnshesqhifohhtvggurdgtohhmqeenucggtffrrghtthgvrhhnpeeffeegfefgkeetudevgfetgfegfedufeejkeekueeiudfhhfekleefleeiffevgfenucffohhmrghinhepqhifohhtvggurdgtohhmnecukfhppedugeelrdejvddrheefrddvnecuvehluhhsthgvrhfuihiivgeptdenucfrrghrrghmpehinhgvthepudegledrjedvrdehfedrvddphhgvlhhopehojedrphhtrhefgedtrdhqfihothgvugdrtghomhdpmhgrihhlfhhrohhmpegsohhunhgtvghsodegjedtuddtleeliedqfedviegrqdhprghvvghlshdrghhurhhskhhishepihhthhgvshhiohhnrdgtohhmsegvmhegieeijedrqhifohhtvggurdgtohhmpdhmohguvgepshhmthhppdhgvghtqdfurghfvggfnhhsuhgsshgtrhhisggvpdhgvghokffrpehushdpshhpfhepphgrshhspdgukhhimhepphgrshhspdgumhgrrhgtpehprghsshdprhgvvhfkrfepohejrdhpthhrfeegtddrqhifohhtvggurdgtohhmrddplh gvrghrnhhinhhgpehtrhhuvgdpnhgspghrtghpthhtohepud",
      "x-vadesecure-score": "17",
      "x-vadesecure-sid": "1596227a-18820b6984d37d51",
      "x-vadesecure-dom": "oxcloud-pro-eu-1",
      "authentication-results": "oxseu-vadesecure.net; iprev=pass policy.iprev=149.72.53.2; spf=pass (sender IP is 149.72.53.2) smtp.mailfrom=bounces@em4667.sender.com; dkim=pass header.d=sender.com header.s=qr header.i=@sender.com; dmarc=pass action=quarantine header.from=sender.com; arc=none;",
      "x-ox-dmarc": "pass (policy=quarantine)",
      "received-dkim": "Success",
      "received-spf": "Pass",
      "dkim-signature": "v=1; a=rsa-sha256; c=relaxed/relaxed; d=sender.com;\th=content-type:date:from:mime-version:subject:to:cc:content-type:date:\tfrom:subject:to;\ts=qr; bh=xQT/zOiSQrLSF8/6EEMnZ45643FAuntwXuLn9D5pQaU=;\tb=tt0lJ7R/XtV+gZxNSsDCCTiUZgbJ1i+mIEMS6jZbiBnZwlKkdUQQx0k8YhpzQt8z75Rw\txTUKgsXK4qda8VG1usIs9Sc22zkWbhPt2sBSFNp/ttRWth7ykxSob4bIx3VOilYuUwux/a\tZM9ciwD47MciWyYXoupOnEuH0z5r3kXsj9U+CVC6rG3VMYX9p+0Zevn34MEabWznWaf/2u\tvhhdF4lLVMGas8NK8iroOboWt/OeTpN5jUfbsKkxoQz55/IU8BejMn5EEyy+Z/wFxe7kWG\t3xjI1z0jPYofWg+0I7o9caVyezuVKokFDK0IWdVeDeDbM458qyoCuIl3X0uvbF0w==",
      "content-type": "multipart/alternative; boundary=\"b0ac8fffccb6a8f9f0e8c1c631323db8238bcba497cedb7766617effed03\"",
      "date": "Wed, 17 Dec 2025 15:49:11 +0000",
      "from": "Sender <notifications@sender.com>",
      "mime-version": "1.0",
      "message-id": "<dsIL2Z9xTiKRffmacoohqw@geopod-ismtpd-10>",
      "subject": "Subject",
      "x-sg-eid": "u001.HFBY8jXZ6FFpjCPBTbWxQ5mM7TS/BoPnr/lfXOQhk3qb2N1+/NKv9m8wrZ2EuXhTz3mRovpxvYwcVIV8EQY59TiT+RTgV6GEbIOD9MWf2EdzMpyQ/grhafZJTjjat/xpZh8s9Vn53VMQm8awqnSbvsoADhzgGXc16e6v4Maq3Wy7dTpqj+yRzwlow/ZaC1UkHUPYrZsTxvDPnm2oPXnYi3ZRYkkvX2vo2pTl42Nny6ibWsxMcD3POKc0e26ibgic",
      "x-sg-id": "u001.SdBcvi+Evd/bQef8eZF3BpTL9BgbK5wfSJMJGMsmprBFxBWXHx8IgIwhzQOJPmSGz+foNC2Amgn8CuSZu4OgaXIIfMtF+riF/99UUtPycuI=",
      "to": "recipient.name@recipient.com",
      "x-entity-id": "u001.7BTMEJwsjztbp2Bukct17g=="
    }
  },
  "body": {
    "attachments": [],
    "text": "Message Text",
    "html": "<!DOCTYPE html PUBLIC \"-//W3C//DTD XHTML 1.0 Transitional//EN\" \"http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd\">\r\n<html xmlns=\"http://www.w3.org/1999/xhtml\" lang=\"en\">\r\n  <head>\r\n    <title>Sender</title>\r\n    <meta http-equiv=\"Content-Type\" content=\"text/html charset=UTF-8\">\r\n    <style type=\"text/css\">\r\n  /* GENERAL STYLE RESETS */\r\n  body, #bodyTable {\r\n    height: 100% !important;\r\n    width: 100% !important;\r\n    margin: 0;\r\n    padding-left: 10px;\r\n    padding-right: 10px;\r\n  }\r\n\r\n  img, a img {\r\n    border: 0;\r\n    outline: none;\r\n    text-decoration: none;\r\n    max-width: 100%;\r\n  }\r\n\r\n  .imageFix {\r\n    display: block;\r\n  }\r\n\r\n  table, td {\r\n    border-collapse: collapse;\r\n  }\r\n\r\n  /* CLIENT-SPECIFIC RESETS */\r\n  table, td {\r\n    mso-table-lspace: 0pt;\r\n    mso-table-rspace: 0pt;\r\n  }\r\n\r\n  img {\r\n    -ms-interpolation-mode: bicubic;\r\n  }\r\n\r\n  body, table, td, p, a, li, blockquote {\r\n    -ms-text-size-adjust: 100%;\r\n    -webkit-text-size-adjust: 100%;\r\n  }\r\n</style>\r\n\r\n\r\n\r\n      <style>\r\n    a {\r\n      text-decoration: none;\r\n    }\r\n\r\n    a:hover {\r\n      text-decoration: underline;\r\n    }\r\n\r\n    @media screen and (max-width: 600px) {\r\n      .two-column {\r\n        display: block !important;\r\n        width: 100% !important;\r\n        max-width: 100% !important;\r\n      }\r\n\r\n      .two-column h2, .two-column h3 {\r\n        margin-left: 0 !important;\r\n      }\r\n\r\n      .two-column h3 {\r\n        margin-bottom: 0.7em !important;\r\n      }\r\n\r\n      .two-column center {\r\n        text-align: left !important;\r\n        margin-bottom: 0.4em !important;\r\n      }\r\n    }\r\n  </style>\r\n\r\n  </head>\r\n  <body width=\"100%\" style=\"margin: 0; mso-line-height-rule: exactly;\">...</body>\r\n</html>"
  },
  "meta": {
    "source": "imap",
    "raw_size_bytes": 26546,
    "received_at": "2025-12-17T15:49:14Z"
  }
}
```
