DECLARE @html1 nvarchar(MAX);
DECLARE @html2 nvarchar(MAX);
DECLARE @html nvarchar(MAX);
DECLARE @bbb  nvarchar(MAX);
DECLARE @bb   nvarchar(MAX);
DECLARE @b   nvarchar(MAX);
declare @a  nvarchar(MAX);
DECLARE @CRLF char(2);

drop table ##MONITOR_OSR
drop table ##monitor_count 

SELECT m.marketer_nme AS MARKETER
,m.city_nme AS CITY
,i.sap_invoice_nbr AS SAP_INVOICE_NUMBER
,i.cst_petro_id AS CUSTOMER
,i.supply_pnt_nbr AS SUPPLY_POINT
,i.idoc_nbr AS IDOC_NUMBER
,i.idoc_id AS IDOC_ID
,i.doc_type AS IDOC_TYPE
,i.received_dte AS IDOC_RECEIVED_DATE
,CASE WHEN (error_desc LIKE '%web service%') THEN 'OSRSUPPORTTEAM' 
	WHEN (error_desc LIKE '%discrepancy%') THEN 'WEBMETHODS\SAP'
	WHEN (error_desc IS NULL) THEN 'NOT YET PICKED UP FOR PROCESSING'
	ELSE 'OSR_SUPPORT_TEAM' END AS ACTION_ITEM_FOR
,lg.error_desc AS ERROR_DESCRIPTION 
into  ##MONITOR_OSR
FROM idoc i 
LEFT OUTER JOIN supplypnt sp ON sp.supplypnt_nbr=i.supply_pnt_nbr
LEFT OUTER JOIN marketer m ON m.supplypnt_id=sp.supplypnt_id
LEFT OUTER JOIN tbl_errors_log lg ON lg.document_id=i.idoc_id
WHERE i.process_flg=0 
ORDER BY m.city_nme,received_dte, i.doc_type DESC


SELECT m.marketer_nme as marketer,m.city_nme as city,COUNT(m.marketer_nme) as count
into  ##MONITOR_count
FROM idoc i 
INNER JOIN supplypnt sp ON sp.supplypnt_nbr=i.supply_pnt_nbr
INNER JOIN marketer m ON m.supplypnt_id=sp.supplypnt_id
LEFT OUTER JOIN tbl_errors_log lg ON lg.document_id=i.idoc_id
WHERE i.process_flg=0 
GROUP BY m.city_nme,m.marketer_nme



EXEC spQueryToHtmlTable @html = @html OUTPUT,  @query = N'select * from  ##MONITOR_OSR';
--EXEC spQueryToHtmlTable @html = @html OUTPUT,  @query = N'select * from  ##MONITOR_count' ;
DECLARE @EMAIL_SENDMAIL_PROFILE_NAME NVARCHAR (MAX)
DECLARE @EMAIL_ADDRESSES NVARCHAR(MAX) 
DECLARE @EMAIL_SUBJECT NVARCHAR (MAX)
DECLARE @EMAIL_BODY NVARCHAR (1000) 
DECLARE @EMAIL_FILE_NAME NVARCHAR (MAX)

SET @EMAIL_SENDMAIL_PROFILE_NAME= 'OSR Monitor'
SET @EMAIL_ADDRESSES ='adeepananda@suncor.com'
SET @EMAIL_SUBJECT='SERVER: '+@@SERVERNAME+'  Alert OSR monitoring list-CAUTION:Donot reply.This email has been generated automatically by job.'
set @bbb=CONCAT('Hi All,',
'<br>',
'<br>',
'Below is a summary of the unprocessed IDOCs and backup documents.',
'<br>',' Please find below details.','<br>','<br>')+ @html

EXEC spQueryToHtmlTable @html = @html OUTPUT,  @query = N'select * from  ##MONITOR_count' ;
set @bb=@bbb+CHAR(13)+CONCAT('<br>','Total count of unprocessed Idocs with respect to each Marketer ',@html,'<br>')

--EXEC spQueryToHtmlTable @html = @html OUTPUT,  @query =N'EXEC sr_GetAllFtpDocScanImagesToWebMethods 1, 1000';
--set @b=@bb+@html
--set @b='EXEC sr_GetAllFtpDocScanImagesToWebMethods 1, 1000'

EXEC sr_GetAllFtpDocScanImagesToWebMethods 1, 1000
set @a= @@rowcount;
set @b=@bb+'  '+'Total Count of Backup Documents Backlog- '+CHAR(13)+@a +CONCAT('<br>','<br>','Thank you,',
'<br>','Deepananda arthi.S',
'<br>')

EXEC msdb.dbo.sp_send_dbmail
   @profile_name=@EMAIL_SENDMAIL_PROFILE_NAME,
@recipients=@EMAIL_ADDRESSES,
@subject=@EMAIL_SUBJECT,
    @body = @b,
   @body_format = 'HTML';
   -- @query_no_truncate = 1,
   -- @attach_query_result_as_file = 1;
	 -- @query = 'SELECT * from ##MONITOR_OSR; select * from ##MONITOR_count';

	  --exec msdb.dbo.sysmail_configure_sp 'MaxFileSize','2000000'-- 2GB

