---
title: "ZLER0038 → CDS View Entity 재설계 분석"
categories:
  - sap
  - abap
tags:
  - CDS
  - ABAP
  - SAP
  - LE
---

## 개요

`ZLER0038`은 **[LE] 베스틸 출하의뢰 잔량** 조회 프로그램이다.  
이 포스트에서는 해당 프로그램의 Selection-Screen 조건, SELECT 구문, 최종 출력 구조를 분석하여  
동등한 기능을 수행하는 **CDS View Entity** 설계안을 도출한다.

---

## 1. Selection-Screen 조건 분석

| 파라미터 | 기준 필드 | 필수 | 설명 |
|---|---|---|---|
| `S_WERKS` | `LIPS-WERKS` | ✅ | 플랜트 |
| `S_WADAT` | `LIKP-WADAT` | ✅ | 계획출고일 (기본값: 오늘) |
| `S_LGORT` | `LIPS-LGORT` | | 저장위치 |
| `S_LFART` | `LIKP-LFART` | | 납품유형 |
| `S_SDABW` | `LIKP-SDABW` | | 특수처리 |
| `S_KUNZA` | `LIKP-KUNNR` | | 의뢰자 (영업담당자) |
| `S_EMPNM` | `ZHRT0010-EMP_NM` | | 의뢰자 이름 |
| `S_KUNNR` | `LIKP-KUNNR` | | 납품처 |
| `S_KUNAG` | `LIKP-KUNAG` | | 판매처 |
| `S_NAMAG` | `KNA1-NAME1` | | 판매처명 (패턴 검색) |
| `S_FEVOR` | `MARC-FEVOR` | | 생산감독자 |
| `S_VKBUR` | `LIPS-VKBUR` | | 영업오피스 |
| `S_CATE` | `MARA-ZZ_ITEMCATE` | | 품목종류 |
| `S_MVGR4` | `MVKE-MVGR4` | | 제품분류 |
| `S_PONLT` | `LIPS-ZZ_PON_LOT` | | PON/LOT 번호 |
| `S_NAMEZB` | `KNA1-NAME1` | | 실수요가명 |
| `S_TRSTA` | `LIKP-TRSTA` | | 운송상태 |
| `S_WBSTK` | `LIKP-WBSTK` | | 출고상태 (숨김) |
| `P_RAD1~4` | - | | 납품유형 라디오 버튼 그룹 |
| `P_EXP1~3` | - | | 수출유형 라디오 버튼 그룹 |

---

## 2. 사용 테이블 및 JOIN 구조

### 2-1. 메인 드라이빙 JOIN (LT_DATA)

```sql
FROM LIKP AS A                          -- 납품 헤더
  INNER JOIN LIPS AS B                  -- 납품 항목
    ON A~VBELN = B~VBELN
  LEFT OUTER JOIN KNA1 AS D             -- 납품처 고객 마스터
    ON A~KUNNR = D~KUNNR
  LEFT OUTER JOIN MARC AS E             -- 자재 플랜트 데이터
    ON B~WERKS = E~WERKS AND B~MATNR = E~MATNR
  LEFT OUTER JOIN VBAP AS F             -- 판매오더 항목
    ON B~VBELV = F~VBELN AND B~POSNV = F~POSNR
  LEFT OUTER JOIN TZONT AS G            -- 시간대 텍스트
    ON D~LZONE = G~ZONE1
  LEFT OUTER JOIN TVSAKT AS H           -- 특수처리 텍스트
    ON H~SDABW = A~SDABW AND H~SPRAS = SY-LANGU
  LEFT OUTER JOIN KNA1 AS KNA1_AG       -- 판매처 고객 마스터
    ON A~KUNAG = KNA1_AG~KUNNR
  LEFT OUTER JOIN VBPA AS VBPA          -- 비즈니스 파트너 (ZA: 의뢰자)
    ON A~VBELN = VBPA~VBELN
    AND VBPA~POSNR = '000000'
    AND VBPA~PARVW = 'ZA'
  LEFT OUTER JOIN MVKE AS MVKE          -- 자재 판매 데이터
    ON MVKE~MATNR = B~MATNR
    AND MVKE~VKORG = A~VKORG
    AND MVKE~VTWEG = B~VTWEG
  LEFT OUTER JOIN TVM4T AS TVM4T        -- 제품분류 텍스트
    ON TVM4T~MVGR4 = MVKE~MVGR4
```

### 2-2. 핵심 WHERE 조건 (필터링 로직)

```sql
WHERE B~WERKS IN S_WERKS
  AND A~VSTEL NOT IN ('2900', '3900')
  AND ( (A~LFART LIKE '%LF' AND A~ZZ_EXPTYPE = ' ')
     OR (A~LFART LIKE '%LF' AND A~ZZ_EXPTYPE <> ' ' AND A~INCO1 = 'EXW')
     OR (A~LFART LIKE '%NL')
     OR (A~LFART LIKE '%LO') )
  AND A~WADAT IN S_WADAT
  AND B~LGORT IN S_LGORT
  AND A~KUNNR IN S_KUNNR
  AND A~KUNAG IN S_KUNAG
  AND A~SDABW IN S_SDABW
  AND B~SPART IN RA_SPART          -- 사업부(30,40 제외)
  AND E~FEVOR IN S_FEVOR
  AND B~VKBUR IN S_VKBUR
  AND A~LFART IN RA_LFART
  AND A~LFART <> 'ZLR'
  AND A~TCODE IN RA_TCODE
  AND A~TRSTA IN S_TRSTA
  AND A~WBSTK IN S_WBSTK
  AND A~ZZ_EXPTYPE IN RA_EXPTY
  AND B~LFIMG > 0
  AND B~ZZ_PON_LOT IN S_PONLT
  AND A~LIFSK = ' '                -- 납품 블록 없음
  AND KNA1_AG~NAME1 IN LR_NAME1    -- 판매처명 패턴 필터
  AND VBPA~KUNNR IN S_KUNZA        -- 의뢰자 필터
  AND MVKE~MVGR4 IN S_MVGR4
```

### 2-3. 부가 조회 테이블

| 테이블 | 목적 | 조인 키 |
|---|---|---|
| `ZPPT9000` + `MSKA` | 판매오더 배정재고 (KALAB) | `MATNR`, `CHARG`, `WERKS` |
| `ZPPT9000` + `MCHB` | 배치별 재고 (CLABS, 대체) | `MATNR`, `CHARG`, `WERKS` |
| `ZWMT1200` | 스펙트로(S-대상) 정보 | `WERKS`, `ZFLAG` |
| `ZCAT0102` | 단면색상 코드명 (`SD3018`) | `CD`, `CD_CLF`, `SPRAS` |
| `MARA` + `ZMDMT0500` | 자재 마스터 + 품목종류명 | `MATNR`, `ZZ_ITEMCATE` |
| `ZCAT0101`+`ZCAT0102` | 사내강종코드명 (`MDM1004`) | `CD`, `CD_CLF`, `SPRAS` |
| `ZWMT1250` | 출하의뢰량(내수/수출/내륙) | `VBELN`, `ZZ_PON_LOT` |
| `ZLET_OFF` | 출하의뢰량(오프라인) | `VBELN`, `ZZ_PON_LOT` |
| `ZSDTLIPS` | 최초의뢰량 (`ORMNG`) | `VBELN`, `ZZ_PON_LOT` |
| `VBPA` + `ZHRT0010` | 의뢰자 부서/이름 (`ZA`) | `VBELN`, `PARVW` |
| `VBAK` | 판매오더 헤더 (Part No.) | `VBELN` |
| `ZPPT1061` | 생산의뢰번호 | `WERKS`, `VBELN`, `POSNR` |
| `VTTP` + `VTTK` | 운송 정보 (`ADD03`) | `VBELN`, `TKNUM` |

---

## 3. 계산 로직 (CASE / 파생 필드)

| 필드 | 계산 방식 |
|---|---|
| `ZZ_BAECHA_TXT` | `CASE ZZ_BAECHA WHEN 'D01' → '당사배차' WHEN 'D02' → '고객사배차'` |
| `ZZ_TRAILER_TXT` | `CASE ZZ_TRAILER WHEN 'A01' → '가능' WHEN 'A02' → '불가능' WHEN 'A03' → '무관'` |
| `ZZ_STACKING_TXT` | `CASE ZZ_STACKING WHEN 'B01' → '2단' WHEN 'B02' → '1단' WHEN 'B03' → '무관'` |
| `ZZ_RAIN_PACKING_TXT` | `CASE ZZ_RAIN_PACKING WHEN 'C01' → '필요' WHEN 'C02' → '불필요' WHEN 'C03' → '무관'` |
| `ZZ_SAT_LANDING_TXT` | `CASE ZZ_SAT_LANDING WHEN 'D01' → '가능' WHEN 'D02' → '불가능' WHEN 'D03' → '사전확인요'` |
| `ZZ_START_CALL_TXT` | `CASE ZZ_START_CALL WHEN 'E01' → '필요' WHEN 'E02' → '불필요' WHEN 'E03' → '무관'` |
| `LFIMG_NEC` (잔량) | `ORMNG - LFIMG_EQC` (최초의뢰량 - 출하완료량) |
| `KALAB` | `SUM(MSKA-KALAB)` 또는 `SUM(MCHB-CLABS)` (형단조 여부에 따라) |
| `ZZ_GRADETYP` | 납품유형 `ZLO`이면 `MARA-ZZ_GRADETYP`, 그 외 `VBAP-ZZ_GRADETYP` |
| `MVGR4_NM` | `COALESCE(TVM4T-BEZEI, ' ')` |

---

## 4. 최종 출력 구조 (GS_HEAD → CDS 컬럼 매핑)

| CDS 필드명 | 소스 테이블/필드 | 설명 |
|---|---|---|
| `OrigDeliveryNo` | `LIKP-VBELN` (분할 원본) | 원납품번호 |
| `DeliveryNo` | `LIKP-VBELN` | 납품번호 |
| `DeliveryType` | `LIKP-LFART` | 납품유형 |
| `SalesClosedFlag` | `LIPS-ZZ_DUMMY1` | 영업종결 여부 |
| `PlannedGIDate` | `LIKP-WADAT` | 계획출고일 |
| `CreatedTime` | `LIKP-ERZET` | 생성시간 |
| `DeliveryDate` | `LIKP-LFDAT` | 납품일 |
| `CreatedDate` | `LIKP-ERDAT` | 생성일 |
| `ShipToParty` | `LIKP-KUNNR` | 납품처 |
| `GoodsMovementStatus` | `LIKP-WBSTK` | 출고상태 |
| `ShipToName` | `KNA1-NAME1` | 납품처명 |
| `ShipToZone` | `KNA1-LZONE` | 배달구역 |
| `ZoneText` | `TZONT-VTEXT` | 구역 텍스트 |
| `ShipToCity` | `KNA1-ORT01` | 도시 |
| `SoldToParty` | `KNA1-KUNNR` (KNA1_AG) | 판매처 |
| `SoldToName` | `KNA1-NAME1` (KNA1_AG) | 판매처명 |
| `EndCustomer` | `KNA1-KUNNR` (ZB) | 실수요가 |
| `EndCustomerName` | `KNA1-NAME1` (ZB) | 실수요가명 |
| `Material` | `LIPS-MATNR` | 자재번호 |
| `ProdSupervisor` | `MARC-FEVOR` | 생산감독자 |
| `Plant` | `LIPS-WERKS` | 플랜트 |
| `StorageLocation` | `LIPS-LGORT` | 저장위치 |
| `HigherLevelItem` | `LIPS-UECHA` | 상위항목 |
| `ReqDelivQty` | `LIPS-LFIMG` (계산) | 출하의뢰량 |
| `OrigOrderQty` | `LIPS-ORMNG` (계산) | 최초의뢰량 |
| `SalesOrderRef` | `LIPS-VBELV` | 참조 판매오더 |
| `PonLot` | `LIPS-ZZ_PON_LOT` | PON/LOT |
| `SteelLength` | `ZPPT9000-ZZ_LENGTH` | 길이 |
| `BundleType` | `LIPS-ZZ_BUNDLE_TYPE` | 번들유형 |
| `ShippingTypeText` | `TVSAKT-BEZEI` | 특수처리 텍스트 |
| `AvailableStock` | `MSKA-KALAB` (집계) | 가용재고 |
| `OrderIrnName` | `VBAP-ZZ_ORD_IRN_NAME` | 주문사양명 |
| `GradeType` | `VBAP/MARA-ZZ_GRADETYP` | 강종코드 |
| `GradeTypeName` | `ZCAT0102-CDNM` | 강종코드명 |
| `ItemType` | `VBAP-ZZ_ITEMTYPE` | 품목유형 |
| `ShapeType` | `VBAP-ZZ_SHAPETYP` | 형상유형 |
| `Material2` | `VBAP-ZZ_MATERIAL` | 소재 |
| `Surface` | `VBAP-ZZ_SURFACED` | 표면처리 |
| `HeatTreatment` | `VBAP-ZZ_HEATTREA` | 열처리 |
| `SalesOffice` | `LIPS-VKBUR` | 영업오피스 |
| `ReceiverName` | `LIKP-ZZ_REC_NAME` | 수령인 |
| `ReceiverTel` | `LIKP-ZZ_REC_TEL` | 수령인 연락처 |
| `ShipmentNo` | `LIKP-ZZ_SMNO` | 배차번호 |
| `VehicleAssignTxt` | CASE `LIKP-ZZ_BAECHA` | 배차방법 텍스트 |
| `TrailerTxt` | CASE `LIKP-ZZ_TRAILER` | 트레일러 텍스트 |
| `StackingTxt` | CASE `LIKP-ZZ_STACKING` | 적재방법 텍스트 |
| `RainPackingTxt` | CASE `LIKP-ZZ_RAIN_PACKING` | 우천포장 텍스트 |
| `SatLandingTxt` | CASE `LIKP-ZZ_SAT_LANDING` | 토요양하 텍스트 |
| `StartCallTxt` | CASE `LIKP-ZZ_START_CALL` | 출발연락 텍스트 |
| `DoTel1` | `LIKP-ZZ_DO_TEL1` | D/O 연락처1 |
| `DoTel2` | `LIKP-ZZ_DO_TEL2` | D/O 연락처2 |
| `DoTel3` | `LIKP-ZZ_DO_TEL3` | D/O 연락처3 |
| `ExportType` | `LIKP-ZZ_EXPTYPE` | 수출유형 |
| `ShippingCond` | `LIKP-ZZ_SHCON` | 해운조건 |
| `ShippingPort` | `LIKP-ZZ_SHPORT` | 해운항 |
| `ProductGroup4` | `MVKE-MVGR4` | 제품분류 |
| `ProductGroup4Name` | `TVM4T-BEZEI` | 제품분류명 |
| `PartNo` | `VBAK-ZZ_PART_NO` | Part Number |
| `SpectroTarget` | `ZWMT1200` 조회 결과 | S-대상 여부 |
| `HeatNo` | `ZPPT9000-ZZ_HEAT` | 히트번호 |
| `MaterialSpec` | `MARA-GROES` | 규격 |
| `ShippedQty` | `SUM(LIPS-LFIMG)` (WBSTK='C') | 출하완료량 |
| `RemainingQty` | `ORMNG - ShippedQty` | 잔량 |
| `RequesterDept` | `ZHRT0010-DEPT_NAME` | 의뢰부서 |
| `RequesterName` | `ZHRT0010-EMP_NM` | 의뢰자명 |
| `SectionColorCode` | `VBAP-ZZ_SECT_CLR` | 단면색상 코드 |
| `SectionColorName` | `ZCAT0102-CDNM` | 단면색상명 |
| `BundleQty` | `LIPS-ZZ_BUNDLE_QTY` | 번들수량 |
| `ItemCategory` | `MARA-ZZ_ITEMCATE` | 품목종류 코드 |
| `ItemCategoryName` | `ZMDMT0500-ITEMCATE_NM` | 품목종류명 |
| `CustomerPartNo` | `MARA-ZZ_CUSTPANO` | 업체부품번호 |
| `LongDescription` | `MARA-ZZ_LONGDESC` | 품목명 |
| `PartType` | `MARA-ZZ_PARTTYPE` | 품목규격 코드 |
| `PartTypeName` | `ZCAT0102-CDNM` | 품목규격명 |
| `ProdOrderReqNo` | `ZPPT1061-PRDREQ` | 생산의뢰번호 |
| `SalesOrderItem` | `VBAP-POSNR` | 판매오더 항목 |

---

## 5. CDS View Entity 설계안

### 5-1. 기본 뷰 — `ZI_LE_DeliveryRemain`

메인 INNER JOIN과 LEFT OUTER JOIN을 CDS Associations 또는 인라인 조인으로 구현한다.

```cds
@AbapCatalog.sqlViewName: 'ZVI_LE_DLVREMN'
@AbapCatalog.compiler.compareFilter: true
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: '베스틸 출하의뢰 잔량 기본뷰'

define view entity ZI_LE_DeliveryRemain
  as select from likp as Header
    inner join   lips as Item
      on Header.vbeln = Item.vbeln
    left outer join kna1 as ShipTo
      on Header.kunnr = ShipTo.kunnr
    left outer join marc as PlantMat
      on  Item.werks = PlantMat.werks
      and Item.matnr = PlantMat.matnr
    left outer join vbap as SalesItem
      on  Item.vbelv = SalesItem.vbeln
      and Item.posnv = SalesItem.posnr
    left outer join tzont as ZoneText
      on ShipTo.lzone = ZoneText.zone1
    left outer join tvsakt as ShipCond
      on  ShipCond.sdabw = Header.sdabw
      and ShipCond.spras = $session.system_language
    left outer join kna1 as SoldTo
      on Header.kunag = SoldTo.kunnr
    left outer join mvke as MatSales
      on  MatSales.matnr = Item.matnr
      and MatSales.vkorg = Header.vkorg
      and MatSales.vtweg = Item.vtweg
    left outer join tvm4t as ProdGrp4T
      on ProdGrp4T.mvgr4 = MatSales.mvgr4
{
  key Header.vbeln                    as DeliveryNo,
  key Item.posnr                      as DeliveryItem,

      -- 헤더 기본 정보
      Header.lfart                    as DeliveryType,
      Header.wadat                    as PlannedGIDate,
      Header.lfdat                    as DeliveryDate,
      Header.erdat                    as CreatedDate,
      Header.erzet                    as CreatedTime,
      Header.kunnr                    as ShipToParty,
      Header.kunag                    as SoldToParty,
      Header.wbstk                    as GoodsMovementStatus,
      Header.trsta                    as TransportationStatus,
      Header.sdabw                    as SpecialProcessing,
      Header.vstel                    as ShippingPoint,
      Header.inco1                    as IncotermsClassif,
      Header.vkorg                    as SalesOrg,
      Header.lifsk                    as DeliveryBlock,

      -- 납품 항목
      Item.matnr                      as Material,
      Item.werks                      as Plant,
      Item.lgort                      as StorageLocation,
      Item.charg                      as Batch,
      Item.vbelv                      as SalesOrderRef,
      Item.posnv                      as SalesOrderItem,
      Item.lfimg                      as DelivQty,
      Item.ormng                      as OrigOrderQty,
      Item.uecha                      as HigherLevelItem,
      Item.vkbur                      as SalesOffice,
      Item.vtweg                      as DistributionChannel,
      Item.spart                      as Division,
      Item.vgtyp                      as PredecessorDocCat,
      Item.zz_pon_lot                 as PonLot,
      Item.zz_bundle_type             as BundleType,
      Item.zz_bundle_qty              as BundleQty,

      -- 납품처 정보
      ShipTo.name1                    as ShipToName,
      ShipTo.lzone                    as ShipToZone,
      ShipTo.ort01                    as ShipToCity,

      -- 판매처 정보
      SoldTo.name1                    as SoldToName,

      -- 구역 텍스트
      ZoneText.vtext                  as ZoneText,

      -- 특수처리 텍스트
      ShipCond.bezei                  as ShippingCondText,

      -- 플랜트/자재
      PlantMat.fevor                  as ProdSupervisor,

      -- 판매오더 항목 확장 필드
      SalesItem.zz_ord_irn_name       as OrderIrnName,
      SalesItem.zz_gradetyp           as GradeType,
      SalesItem.zz_itemtype           as ItemType,
      SalesItem.zz_itemcate           as ItemCategoryFromOrder,
      SalesItem.zz_shapetyp           as ShapeType,
      SalesItem.zz_material           as MaterialSpec,
      SalesItem.zz_surfaced           as Surface,
      SalesItem.zz_heattrea           as HeatTreatment,
      SalesItem.zz_sect_clr           as SectionColorCode,

      -- 제품분류
      MatSales.mvgr4                  as ProductGroup4,
      coalesce( ProdGrp4T.bezei, '' ) as ProductGroup4Name,

      -- 배차 관련 CASE 변환
      case Header.zz_baecha
        when 'D01' then '당사배차'
        when 'D02' then '고객사배차'
        else ''
      end                             as VehicleAssignText,

      case Header.zz_trailer
        when 'A01' then '가능'
        when 'A02' then '불가능'
        when 'A03' then '무관'
        else ''
      end                             as TrailerText,

      case Header.zz_stacking
        when 'B01' then '2단'
        when 'B02' then '1단'
        when 'B03' then '무관'
        else ''
      end                             as StackingText,

      case Header.zz_rain_packing
        when 'C01' then '필요'
        when 'C02' then '불필요'
        when 'C03' then '무관'
        else ''
      end                             as RainPackingText,

      case Header.zz_sat_landing
        when 'D01' then '가능'
        when 'D02' then '불가능'
        when 'D03' then '사전확인요'
        else ''
      end                             as SatLandingText,

      case Header.zz_start_call
        when 'E01' then '필요'
        when 'E02' then '불필요'
        when 'E03' then '무관'
        else ''
      end                             as StartCallText,

      -- 기타 헤더 확장 필드
      Header.zz_rec_name              as ReceiverName,
      Header.zz_rec_tel               as ReceiverTel,
      Header.zz_smno                  as ShipmentNo,
      Header.zz_do_tel1               as DoTel1,
      Header.zz_do_tel2               as DoTel2,
      Header.zz_do_tel3               as DoTel3,
      Header.zz_exptype               as ExportType,
      Header.zz_shcon                 as ShippingCond,
      Header.zz_shport                as ShippingPort

}
where Item.lfimg > 0
  and Header.lifsk = ''
  and Header.vstel not in ('2900', '3900')
  and Header.lfart <> 'ZLR'
```

### 5-2. 파트너 뷰 — `ZI_LE_DeliveryPartner`

`VBPA`, `KNA1`, `ZHRT0010`를 활용하여 파트너(실수요가, 판매처 담당자) 정보를 분리한다.

```cds
@AbapCatalog.sqlViewName: 'ZVI_LE_DLVPTNR'
@EndUserText.label: '출하의뢰 파트너 정보'

define view entity ZI_LE_DeliveryPartner
  as select from vbpa
    left outer join kna1 as PartnerCustomer
      on vbpa.kunnr = PartnerCustomer.kunnr
    left outer join zhrt0010 as Employee
      on vbpa.kunnr = Employee.partner
{
  key vbpa.vbeln                  as SalesOrderRef,
  key vbpa.parvw                  as PartnerFunction,
      vbpa.kunnr                  as PartnerNo,
      PartnerCustomer.name1       as PartnerName,
      Employee.dept_name          as DepartmentName,
      Employee.emp_nm             as EmployeeName
}
where vbpa.posnr = '000000'
```

### 5-3. 자재 마스터 뷰 — `ZI_LE_MaterialInfo`

품목종류, 품목규격, 강종 정보를 `MARA`, `ZMDMT0500`, `ZCAT0102`, `ZCAT0101`로 구성한다.

```cds
@AbapCatalog.sqlViewName: 'ZVI_LE_MATLNFO'
@EndUserText.label: '출하의뢰 자재 마스터 정보'

define view entity ZI_LE_MaterialInfo
  as select from mara
    left outer join zmdmt0500 as ItemCateName
      on mara.zz_itemcate = ItemCateName.itemcate
    left outer join zcat0102 as CustomerPartNo
      on  mara.zz_custpano = CustomerPartNo.cd
      and CustomerPartNo.cd_clf = 'MDM1017'
      and CustomerPartNo.spras = $session.system_language
    left outer join zcat0102 as PartTypeName
      on  mara.zz_parttype = PartTypeName.cd
      and PartTypeName.cd_clf = 'MDM1016'
      and PartTypeName.spras = $session.system_language
{
  key mara.matnr                    as Material,
      mara.groes                    as MaterialSpec,
      mara.zz_longdesc              as LongDescription,
      mara.zz_itemcate              as ItemCategory,
      ItemCateName.itemcate_nm      as ItemCategoryName,
      mara.zz_custpano              as CustomerPartNo,
      CustomerPartNo.cdnm           as CustomerPartNoName,
      mara.zz_parttype              as PartType,
      PartTypeName.cdnm             as PartTypeName,
      mara.zz_gradetyp              as GradeType
}
```

### 5-4. 재고 뷰 — `ZI_LE_StockByLot`

`ZPPT9000`, `MSKA`, `MCHB`를 이용하여 PON/LOT 별 가용재고를 집계한다.

```cds
@AbapCatalog.sqlViewName: 'ZVI_LE_STKLOT'
@EndUserText.label: '출하의뢰 LOT별 가용재고'

define view entity ZI_LE_StockByLot
  as select from zppt9000 as Lot
    inner join mska as SalesStock
      on  Lot.matnr    = SalesStock.matnr
      and Lot.charg    = SalesStock.charg
      and Lot.zz_werks = SalesStock.werks
{
  key SalesStock.werks              as Plant,
  key Lot.matnr                     as Material,
  key Lot.zz_pon_lot                as PonLot,
      @Semantics.quantity.unitOfMeasure: 'BaseUnit'
      sum( SalesStock.kalab )       as AvailableStock,
      Lot.zz_heat                   as HeatNo,
      Lot.zz_length                 as SteelLength,
      Lot.zz_lgpla                  as StorageBin,
      Lot.zz_bundle                 as BundleNo,
      Lot.zz_line_ty                as LineType,
      Lot.zz_stype                  as SteelType,
      Lot.zz_serial                 as SerialNo,
      Lot.zz_net_weight             as NetWeight,
      Lot.zz_qty                    as LotQty
}
where SalesStock.kalab > 0
group by SalesStock.werks, Lot.matnr, Lot.zz_pon_lot,
         Lot.zz_heat, Lot.zz_length, Lot.zz_lgpla,
         Lot.zz_bundle, Lot.zz_line_ty, Lot.zz_stype,
         Lot.zz_serial, Lot.zz_net_weight, Lot.zz_qty
```

### 5-5. 출하의뢰량 뷰 — `ZI_LE_ReqQtyByLot`

`ZWMT1250`(온라인)과 `ZLET_OFF`(오프라인)을 UNION하여 PON/LOT 별 의뢰량을 합산한다.

```cds
@AbapCatalog.sqlViewName: 'ZVI_LE_REQQLOT'
@EndUserText.label: '출하의뢰량 (온라인+오프라인)'

define view entity ZI_LE_ReqQtyByLot
  as select from zwmt1250
{
  key vbeln                         as DeliveryNo,
  key zz_pon_lot                    as PonLot,
      @Semantics.quantity.unitOfMeasure: 'BaseUnit'
      sum( lfimg )                  as ReqDelivQty,
      sum( kcmeng )                 as ReqKcmengQty,
      sum( zz_bundle_qty )          as BundleQtySum,
      cast( 'ONLINE ' as abap.char(7) ) as Source  -- 7자리로 OFFLINE 과 길이 통일
}
where uecha = ''
group by vbeln, zz_pon_lot

union all

select from zlet_off
{
  key vbeln                         as DeliveryNo,
  key zz_pon_lot                    as PonLot,
      sum( lfimg )                  as ReqDelivQty,
      cast( 0 as abap.dec(13,3) )   as ReqKcmengQty,
      cast( 0 as abap.dec(13,3) )   as BundleQtySum,
      cast( 'OFFLINE' as abap.char(7) ) as Source  -- 7자리
}
group by vbeln, zz_pon_lot
```

### 5-6. 최종 소비 뷰 (Consumption View) — `ZC_LE_DeliveryRemain`

위의 기본 뷰들을 조합하여 실제 업무 소비 뷰를 구성한다.  
이 뷰에 파라미터를 선언하여 Selection-Screen 조건을 CDS 레벨에서 처리한다.

```cds
@AbapCatalog.sqlViewName: 'ZVC_LE_DLVREMN'
@AccessControl.authorizationCheck: #CHECK
@EndUserText.label: '베스틸 출하의뢰 잔량 소비뷰'
@VDM.viewType: #CONSUMPTION
@Search.searchable: false

define view entity ZC_LE_DeliveryRemain
  with parameters
    p_werks   : werks_d,       -- 플랜트 (필수)
    p_wadat_s : dats,          -- 계획출고일 시작 (필수)
    p_wadat_e : dats           -- 계획출고일 종료 (필수)
  as select from ZI_LE_DeliveryRemain as Base
    -- 파트너: 판매처 담당자 (ZA)
    left outer join ZI_LE_DeliveryPartner as Requester
      on  Base.SalesOrderRef   = Requester.SalesOrderRef
      and Requester.PartnerFunction = 'ZA'
    -- 파트너: 실수요가 (ZB)
    left outer join ZI_LE_DeliveryPartner as EndCust
      on  Base.SalesOrderRef   = EndCust.SalesOrderRef
      and EndCust.PartnerFunction = 'ZB'
    -- 자재 마스터
    left outer join ZI_LE_MaterialInfo as MatInfo
      on Base.Material = MatInfo.Material
    -- 재고 (LOT별)
    left outer join ZI_LE_StockByLot as Stock
      on  Base.Plant   = Stock.Plant
      and Base.Material = Stock.Material
      and Base.PonLot  = Stock.PonLot
    -- 출하의뢰량
    left outer join ZI_LE_ReqQtyByLot as ReqQty
      on  Base.DeliveryNo = ReqQty.DeliveryNo
      and Base.PonLot     = ReqQty.PonLot
    -- 판매오더 헤더 (Part No.)
    left outer join vbak as SalesOrder
      on Base.SalesOrderRef = SalesOrder.vbeln
    -- 생산의뢰번호
    left outer join zppt1061 as ProdReq
      on  Base.Plant          = ProdReq.werks
      and Base.SalesOrderRef  = ProdReq.vbeln
      and Base.SalesOrderItem = ProdReq.posnr
    -- 강종 코드명 (MDM1004)
    left outer join zcat0102 as GradeTypeName
      on  Base.GradeType       = GradeTypeName.cd
      and GradeTypeName.cd_clf = 'MDM1004'
      and GradeTypeName.spras  = $session.system_language
    -- 단면 색상명 (SD3018)
    left outer join zcat0102 as SectionColorName
      on  Base.SectionColorCode  = SectionColorName.cd
      and SectionColorName.cd_clf = 'SD3018'
      and SectionColorName.spras  = $session.system_language
{
  -- 키
  key Base.DeliveryNo,
  key Base.DeliveryItem,

  -- 납품 헤더 기본
      Base.DeliveryType,
      Base.PlannedGIDate,
      Base.DeliveryDate,
      Base.CreatedDate,
      Base.CreatedTime,
      Base.GoodsMovementStatus,
      Base.TransportationStatus,
      Base.SpecialProcessing,
      Base.ShippingCondText,
      Base.ExportType,
      Base.ShippingCond,
      Base.ShippingPort,

  -- 납품처 / 판매처
      Base.ShipToParty,
      Base.ShipToName,
      Base.ShipToZone,
      Base.ZoneText,
      Base.ShipToCity,
      Base.SoldToParty,
      Base.SoldToName,
      EndCust.PartnerNo                as EndCustomer,
      EndCust.PartnerName              as EndCustomerName,

  -- 자재
      Base.Material,
      Base.Plant,
      Base.StorageLocation,
      Base.Batch,
      Base.PonLot,
      Base.ProdSupervisor,
      Base.SalesOffice,
      Base.Division,

  -- 자재 마스터 확장
      MatInfo.MaterialSpec,
      MatInfo.LongDescription,
      MatInfo.ItemCategory,
      MatInfo.ItemCategoryName,
      MatInfo.CustomerPartNo,
      MatInfo.CustomerPartNoName,
      MatInfo.PartType,
      MatInfo.PartTypeName,

  -- 강종 / 품목 정보
      Base.GradeType,
      GradeTypeName.cdnm               as GradeTypeName,
      Base.ItemType,
      Base.ShapeType,
      Base.MaterialSpec                as MaterialGrade,
      Base.Surface,
      Base.HeatTreatment,
      Base.OrderIrnName,
      Base.SectionColorCode,
      SectionColorName.cdnm            as SectionColorName,

  -- 번들
      Base.BundleType,
      coalesce( ReqQty.BundleQtySum, 0 ) as BundleQty,

  -- 제품분류
      Base.ProductGroup4,
      Base.ProductGroup4Name,

  -- 참조 문서
      Base.SalesOrderRef,
      Base.SalesOrderItem,
      SalesOrder.zz_part_no            as PartNo,
      ProdReq.prdreq                   as ProdOrderReqNo,

  -- 파트너 (의뢰자)
      Requester.PartnerNo              as RequesterNo,
      Requester.DepartmentName         as RequesterDept,
      Requester.EmployeeName           as RequesterName,

  -- 재고
      coalesce( Stock.AvailableStock, 0 ) as AvailableStock,
      Stock.HeatNo,
      Stock.SteelLength,

  -- 수량 계산
      coalesce( ReqQty.ReqDelivQty, 0 ) as ReqDelivQty,
      Base.OrigOrderQty,
      -- 출하완료량은 별도 집계 뷰 또는 Fiori 앱에서 처리 권장
      -- 잔량 = 최초의뢰량 - 출하완료량 (ORMNG - LFIMG_EQC)

  -- 배차 / 운송 텍스트 (CASE 파생)
      Base.VehicleAssignText,
      Base.TrailerText,
      Base.StackingText,
      Base.RainPackingText,
      Base.SatLandingText,
      Base.StartCallText,

  -- 기타 헤더 확장
      Base.ReceiverName,
      Base.ReceiverTel,
      Base.ShipmentNo,
      Base.DoTel1,
      Base.DoTel2,
      Base.DoTel3
}
where Base.Plant        = $parameters.p_werks
  and Base.PlannedGIDate >= $parameters.p_wadat_s
  and Base.PlannedGIDate <= $parameters.p_wadat_e
```

---

## 6. 설계 결정사항 및 주의점

### 6-1. 잔량 계산 위치

원 프로그램에서 **잔량(LFIMG_NEC)** 은 ABAP 루프 내에서 `ORMNG - LFIMG_EQC` 로 계산된다.  
CDS에서는 이를 두 가지 방식으로 처리할 수 있다.

**방법 A: 출하완료량 서브 집계 뷰 추가**

```cds
-- 출하완료 수량 집계 뷰 (WBSTK = 'C' 인 D/O 기준)
define view entity ZI_LE_ShippedQtyByLot
  as select from lips as Item
    inner join likp as Header
      on Item.vbeln = Header.vbeln
{
  key Item.vbeln      as DeliveryNo,
  key Item.zz_pon_lot as PonLot,
      sum( Item.lfimg ) as ShippedQty
}
where Header.wbstk = 'C'
group by Item.vbeln, Item.zz_pon_lot
```

이후 소비 뷰에 조인하고 파생 필드로 선언한다:

```cds
-- 소비 뷰 내 잔량 파생
Base.OrigOrderQty - coalesce( Shipped.ShippedQty, 0 ) as RemainingQty,
```

**방법 B: Fiori Elements / OData 레이어에서 계산**  
CDS는 원시 수량만 노출하고 UI 레이어에서 계산하는 방식으로, 복잡도를 줄일 수 있다.

### 6-2. D/O 분할(Split) 원본 추적

원 프로그램은 `VBSS` 테이블을 재귀 탐색하여 원 D/O 번호를 역추적한다.  
CDS에서는 `vbss` 테이블을 직접 LEFT JOIN으로 추가하고,  
`coalesce( VBSS.vbelv, Base.DeliveryNo )` 패턴으로 원납품번호를 도출한다.

```cds
left outer join vbss as SplitRef
  on Base.DeliveryNo = SplitRef.vbeln
-- 원납품번호
coalesce( SplitRef.vbelv, Base.DeliveryNo ) as OrigDeliveryNo,
```

### 6-3. MSKA vs MCHB 조건부 조회

원 프로그램은 `MSKA`가 없으면 `MCHB`를 조회한다.  
CDS에서는 두 테이블을 LEFT OUTER JOIN으로 모두 포함하고 `coalesce`를 활용한다.

```cds
left outer join ZI_LE_StockByLot as StockMska   -- MSKA 기반
  on  Base.Plant = StockMska.Plant and ...
left outer join ZI_LE_StockMchb as StockMchb    -- MCHB 기반 (별도 뷰)
  on  Base.Plant = StockMchb.Plant and ...

coalesce( StockMska.AvailableStock,
          StockMchb.AvailableStock, 0 )          as AvailableStock,
```

### 6-4. 선택 조건 중 패턴 검색 (`LIKE`)

CDS View는 `LIKE` 연산자를 직접 WHERE 절에 사용할 수 없다.  
`LFART LIKE '%LF'` 와 같은 조건은 소비 뷰에서 `@ObjectModel.filter.transformedApplicable: true` 를 선언하거나,  
Fiori 레이어의 필터 변환 어노테이션 또는 ABAP CDS의 `$extension` 패턴으로 처리한다.  
대안으로 ABAP 클래스에서 CDS 호출 시 추가 필터를 적용하는 방식도 적합하다.

### 6-5. 다국어 텍스트 (`SPRAS`)

CDS의 `$session.system_language` 로 텍스트 테이블(`TVSAKT`, `TZONT`, `TVM4T`, `ZCAT0102`)을 필터링한다.

---

## 7. 뷰 계층 구조 요약

```
ZC_LE_DeliveryRemain          ← 소비 뷰 (파라미터, 최종 필드 정제)
 ├── ZI_LE_DeliveryRemain      ← 인터페이스 뷰 (메인 JOIN, CASE 변환)
 │    ├── LIKP / LIPS
 │    ├── KNA1 (납품처/판매처)
 │    ├── MARC / VBAP
 │    ├── TZONT / TVSAKT
 │    ├── MVKE / TVM4T
 ├── ZI_LE_DeliveryPartner     ← 파트너 정보 (VBPA + KNA1 + ZHRT0010)
 ├── ZI_LE_MaterialInfo        ← 자재 마스터 (MARA + ZMDMT0500 + ZCAT0102)
 ├── ZI_LE_StockByLot          ← LOT별 가용재고 (ZPPT9000 + MSKA)
 ├── ZI_LE_StockMchb           ← LOT별 배치재고 (ZPPT9000 + MCHB, 대체)
 ├── ZI_LE_ReqQtyByLot         ← 의뢰량 합산 (ZWMT1250 UNION ZLET_OFF)
 ├── ZI_LE_ShippedQtyByLot     ← 출하완료량 집계 (LIPS + LIKP, WBSTK='C')
 ├── VBAK                      ← Part No.
 ├── ZPPT1061                  ← 생산의뢰번호
 ├── ZCAT0102 (강종/색상)       ← 코드명 텍스트
```
