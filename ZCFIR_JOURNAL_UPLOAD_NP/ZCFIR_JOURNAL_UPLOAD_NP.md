```abap
REPORT zcfir_journal_upload_np MESSAGE-ID zfi_msg.

INCLUDE zcfir_journal_upload_np_top.

**********************************************************************
** SELECTION SCREEN **
SELECTION-SCREEN BEGIN OF BLOCK b01 WITH FRAME TITLE TEXT-b01.

  SELECTION-SCREEN BEGIN OF BLOCK b08 WITH FRAME  .
    SELECTION-SCREEN   COMMENT /1(75)  TEXT-c01 MODIF ID mg1.
    SELECTION-SCREEN   COMMENT /1(75)  TEXT-c02 MODIF ID mg1.
**SELECTION-SCREEN   COMMENT /1(75)  TEXT-c03 MODIF ID mg1.
  SELECTION-SCREEN END OF BLOCK b08 .

  SELECTION-SCREEN BEGIN OF BLOCK b11 WITH FRAME TITLE TEXT-b11.
*    PARAMETERS    : p_file TYPE rlgrap-filename OBLIGATORY. "CH-1321 -
    PARAMETERS    : p_file TYPE rlgrap-filename MODIF ID fil. "CH-1321 +
  SELECTION-SCREEN END OF BLOCK b11.
  .

  SELECTION-SCREEN BEGIN OF BLOCK b13 WITH FRAME TITLE TEXT-b13.
    PARAMETERS    : p_test AS CHECKBOX DEFAULT 'X'.
    SELECTION-SCREEN SKIP 1.
    PARAMETERS    : p_vari TYPE slis_vari.
  SELECTION-SCREEN END OF BLOCK b13.
SELECTION-SCREEN END OF BLOCK b01.
SELECTION-SCREEN FUNCTION KEY 1.                            "CH-1321 +
**********************************************************************

INCLUDE zcfir_journal_upload_np_forms.


INITIALIZATION.
*
  PERFORM frm_screen_display_by_tcode.

  sscrfields-functxt_01 = icon_export && TEXT-001.
**********************************************************************
AT SELECTION-SCREEN OUTPUT .
  PERFORM frm_screen_display_by_tcode.


** SELECTION SCREEN EVENTS **
AT SELECTION-SCREEN ON VALUE-REQUEST FOR p_file.
  PERFORM frm_open_dialog.

AT SELECTION-SCREEN ON VALUE-REQUEST FOR p_vari.
  PERFORM frm_get_layout.

AT SELECTION-SCREEN.
  PERFORM frm_check_file_exist.
  PERFORM frm_check_fbb1_authorisation.

  PERFORM frm_screen_display_by_tcode.
  PERFORM frm_download_excel.                               "CH-1321 +
**********************************************************************

**********************************************************************
START-OF-SELECTION.
  TRY."C-001 +
      PERFORM frm_housekeeping.
      PERFORM frm_upload_file.
      PERFORM frm_validate_file.
      PERFORM frm_process_file.
    CATCH cx_root INTO DATA(gx_error)."C-001 +
      MESSAGE e000(zfi_msg) WITH gx_error->get_longtext( ). "C-001 +
  ENDTRY."C-001 +
**********************************************************************

**********************************************************************
END-OF-SELECTION.
  PERFORM frm_display_results.
**********************************************************************
```

```abap
"Constant definition
CONSTANTS gc_x VALUE 'X' .
CONSTANTS gc_koart_gl       TYPE koart VALUE 'S'.
CONSTANTS gc_koart_vendor   TYPE koart VALUE 'K'.
CONSTANTS gc_koart_customer TYPE koart VALUE 'D'.
CONSTANTS gc_koart_asset    TYPE koart VALUE 'A'.
CONSTANTS gc_set_accrual_doc_type   TYPE setnamenew VALUE 'ZFI_VAL_ACCRUALGLDOCTYPE'.
CONSTANTS gc_set_zfi_gl999lines     TYPE setnamenew VALUE 'ZFI_GL_999_LINE_CLEARING'.

*
DATA: gv_rc                    TYPE i,
      gv_exist                 TYPE abap_bool,
      gv_dummy,
      gv_sy_tabix_gt_upload    TYPE i,
      gv_upload_lines          TYPE i,
      gv_posnr                 TYPE i,
      gv_file_counter(5)       TYPE n,
      gv_accrual,
      gv_test,
      gv_do_not_process_update,
      gv_do_not_process.

TYPES: BEGIN OF gty_header_upload,
         row_number   TYPE ebelp,
         ledger_group TYPE accounting_principle,
         bukrs        TYPE bukrs,
         bldat        TYPE bldat,
         budat        TYPE budat,
         monat        TYPE monat,
         blart        TYPE blart,
         waers        TYPE waers,
         xblnr        TYPE xblnr,
         bktxt        TYPE bktxt,
       END OF gty_header_upload.
*
DATA  gv_doc_max_line_cnt TYPE i VALUE 900.
DATA gv_currencyamount_amt_doccur TYPE bapidoccur.
DATA gv_clearing_gl TYPE saknr.

DATA gt_set_values_accrual_docs            TYPE STANDARD TABLE OF rgsb4.
DATA gs_set_values_accrual_docs LIKE LINE OF gt_set_values_accrual_docs.

DATA gt_set_zfi_gl999lines             TYPE STANDARD TABLE OF rgsb4.
DATA gs_set_zfi_gl999lines  LIKE LINE OF gt_set_zfi_gl999lines.
*
DATA gs_t001 TYPE t001.
DATA gt_tbsl TYPE TABLE OF tbsl.
DATA gs_tbsl LIKE LINE OF gt_tbsl .   "TYPE tbsl.
DATA gs_tacc_trgt_ldgr TYPE tacc_trgt_ldgr.
*
DATA gt_upload TYPE TABLE OF zcfis_infile.
DATA gs_upload LIKE LINE OF gt_upload.

*--&JDK
DATA gs_header_upload  LIKE LINE OF gt_upload.
DATA gs_header_upload2 TYPE gty_header_upload.
DATA gt_header_upload  LIKE TABLE OF gs_header_upload.

DATA gt_upload_store TYPE TABLE OF zcfis_infile.
DATA gs_upload_store LIKE LINE OF gt_upload_store.
****report
DATA gt_report TYPE TABLE OF zcfis_journal_upload_np.
DATA gs_report LIKE LINE OF gt_report.

*ALV fieldcatlog
DATA gt_fieldcat            TYPE slis_t_fieldcat_alv.

DATA : gt_alv_list_top_of_page TYPE slis_t_listheader,
       gt_alv_list_end_of_list TYPE slis_t_listheader,
       gt_alv_events           TYPE slis_t_event.

*Accounting document
DATA gt_return            TYPE STANDARD TABLE OF bapiret2.
DATA gs_return LIKE LINE OF gt_return.
DATA gs_documentheader    TYPE bapiache09.
DATA gt_accountgl TYPE TABLE OF bapiacgl09.
DATA gs_accountgl LIKE LINE OF gt_accountgl.
DATA gt_accountreceivable	TYPE TABLE OF	bapiacar09.
DATA gs_accountreceivable LIKE LINE OF gt_accountreceivable.
DATA gt_accountpayable    TYPE TABLE OF bapiacap09.
DATA gs_accountpayable LIKE LINE OF gt_accountpayable.
DATA gt_currencyamount    TYPE   TABLE OF bapiaccr09.
DATA gs_currencyamount LIKE LINE OF gt_currencyamount.

"Extension field
DATA gt_extension1  TYPE TABLE OF   bapiacextc.
DATA gs_extension1 LIKE LINE OF gt_extension1.
TABLES: sscrfields.                                         "CH-1321 +

"Used in enhancemetn to create parked document
DATA gs_acc_ext TYPE zcfis_acc_ext4.                        "CH-1321 +
DATA gt_extension2 TYPE TABLE OF bapiparex.                 "CH-1321 +
```

```abap
*&---------------------------------------------------------------------*
*& FORM frm_upload_file
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_housekeeping.
*
*  load all the Posting Keys
  SELECT *
    FROM tbsl
    INTO TABLE gt_tbsl
    WHERE koart IN ( gc_koart_asset, gc_koart_customer, gc_koart_gl, gc_koart_vendor )
  ORDER BY bschl.

  gv_test = p_test.
*  load accruals doc type from set
*
  CLEAR gt_set_values_accrual_docs.
  CALL FUNCTION 'G_SET_GET_ALL_VALUES'
    EXPORTING
      client        = sy-mandt
*     FORMULA_RETRIEVAL           = ' '
*     LEVEL         = 0
      setnr         = gc_set_accrual_doc_type
*     VARIABLES_REPLACEMENT       = ' '
*     TABLE         = ' '
*     CLASS         = ' '
*     NO_DESCRIPTIONS             = 'X'
*     NO_RW_INFO    = 'X'
*     DATE_FROM     = DATE_FROM
*     DATE_TO       = DATE_TO
*     FIELDNAME     = ' '
    TABLES
      set_values    = gt_set_values_accrual_docs
    EXCEPTIONS
      set_not_found = 1
      OTHERS        = 2.
  IF sy-subrc <> 0.
    MESSAGE e036(zfi_msg) WITH gc_set_accrual_doc_type.
* Set & not found

  ENDIF.
*
*Load the contra for multiple documents
  CLEAR gt_set_zfi_gl999lines.

  CALL FUNCTION 'G_SET_GET_ALL_VALUES'
    EXPORTING
      client        = sy-mandt
*     FORMULA_RETRIEVAL           = ' '
*     LEVEL         = 0
      setnr         = gc_set_zfi_gl999lines
*     VARIABLES_REPLACEMENT       = ' '
*     TABLE         = ' '
*     CLASS         = ' '
*     NO_DESCRIPTIONS             = 'X'
*     NO_RW_INFO    = 'X'
*     DATE_FROM     = DATE_FROM
*     DATE_TO       = DATE_TO
*     FIELDNAME     = ' '
    TABLES
      set_values    = gt_set_zfi_gl999lines
    EXCEPTIONS
      set_not_found = 1
      OTHERS        = 2.
  IF sy-subrc <> 0.
    MESSAGE e036(zfi_msg) WITH gc_set_zfi_gl999lines .
* Set & not found

  ENDIF.

  READ TABLE gt_set_zfi_gl999lines INTO gs_set_values_accrual_docs INDEX 1.

  gv_clearing_gl = gs_set_values_accrual_docs-from.
  SHIFT gv_clearing_gl RIGHT DELETING TRAILING ' '.
  OVERLAY gv_clearing_gl  WITH '0000000000'.

  PERFORM frm_check_fbb1_authorisation.


ENDFORM.

*&---------------------------------------------------------------------*
*& FORM frm_upload_file
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_upload_file .

  DATA: lt_raw_data TYPE truxs_t_text_data   .   "TABLE OF truxs_t_text_data.

  CALL FUNCTION 'TEXT_CONVERT_XLS_TO_SAP'
    EXPORTING
*     I_FIELD_SEPERATOR    =
      i_line_header        = gc_x
      i_tab_raw_data       = lt_raw_data
      i_filename           = p_file
    TABLES
      i_tab_converted_data = gt_upload
    EXCEPTIONS
      conversion_failed    = 1
      OTHERS               = 2.

  IF sy-subrc <> 0.
* Implement suitable error handling here
  ENDIF.
*
  DESCRIBE TABLE gt_upload LINES gv_upload_lines.
*
ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_open_dialog
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_open_dialog .

  DATA lt_loc_files TYPE filetable.

  REFRESH: lt_loc_files.
  CALL METHOD cl_gui_frontend_services=>file_open_dialog
    EXPORTING
      window_title            = 'Select File'
**      default_extension       = '*.xls,*.xlsx'
      default_filename        = '*.xl*'
      multiselection          = ' '
    CHANGING
      file_table              = lt_loc_files
      rc                      = gv_rc
    EXCEPTIONS
      file_open_dialog_failed = 1
      cntl_error              = 2
      error_no_gui            = 3
      not_supported_by_gui    = 4.

  IF sy-subrc EQ 0.
    READ TABLE lt_loc_files INTO p_file INDEX 1.
  ENDIF.

ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_check_file_exist
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_check_file_exist .
***BEGIN OF INSERTION CH-1321
  CHECK sy-ucomm = 'ONLI'.
  IF p_file IS INITIAL.
    SET CURSOR FIELD 'P_FILE'.
    MESSAGE e055(00).
  ENDIF.
***END OF INSERTION CH-1321
  DATA lv_file  TYPE string.
  lv_file = p_file.

  CALL METHOD cl_gui_frontend_services=>file_exist
    EXPORTING
      file                 = lv_file
    RECEIVING
      result               = gv_exist
    EXCEPTIONS
      cntl_error           = 1
      error_no_gui         = 2
      wrong_parameter      = 3
      not_supported_by_gui = 4
      OTHERS               = 5.

  IF gv_exist IS INITIAL.
    MESSAGE s001 WITH p_file DISPLAY LIKE 'E'.
**    CLEAR: p_file.
  ENDIF.

ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_get_layout
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_get_layout .
  DATA: ls_vari TYPE disvariant.

  ls_vari-report    = sy-repid.
  ls_vari-username  = sy-uname.

  CALL FUNCTION 'REUSE_ALV_VARIANT_F4'
    EXPORTING
      is_variant    = ls_vari
      i_save        = 'A'
    IMPORTING
      es_variant    = ls_vari
    EXCEPTIONS
      not_found     = 1
      program_error = 2
      OTHERS        = 3.

  IF sy-subrc EQ 0.
    p_vari = ls_vari-variant.
  ENDIF.


ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_validate_file
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_validate_file .


  DATA lv_sy_tabix TYPE sy-tabix.

***  LOOP AT gt_upload INTO gs_upload.
***    lv_sy_tabix = sy-tabix.
****
***    MODIFY gt_upload FROM gs_upload INDEX lv_sy_tabix.
***  ENDLOOP.

  CLEAR gv_do_not_process_update.
  IF p_test NE gc_x.
    gv_test = gc_x.
    PERFORM frm_process_file .
    LOOP AT gt_report TRANSPORTING NO FIELDS WHERE type = 'E'.
      EXIT.
    ENDLOOP.
    IF sy-subrc EQ 0.
      gv_do_not_process_update = gc_x.
    ELSE.
      CLEAR gt_report.
    ENDIF.
    CLEAR gv_test.
  ENDIF.

ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_process_file
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_process_file .

  DATA lv_line_cnt  TYPE i.
  DATA lv_budat(8) TYPE n.
  DATA lv_bldat(8) TYPE n .


  PERFORM frm_check_fbb1_authorisation.

  CLEAR gv_do_not_process.
  CLEAR gv_file_counter.

  CHECK gv_do_not_process_update IS INITIAL.  " there were no errors during the check
  CLEAR gs_documentheader.
*
  LOOP AT gt_upload INTO gs_upload.

    gv_sy_tabix_gt_upload = sy-tabix.

    PERFORM frm_check_dates.

*---&JDK---INSERT
    PERFORM frm_check_header.
*---&JDK---END

***    lv_budat = gs_upload-budat+4(4) && gs_upload-budat(4).
***    lv_bldat = gs_upload-bldat+4(4) && gs_upload-bldat(4).

    IF  NOT (
**********       gs_documentheader-comp_code = gs_upload-bukrs AND
**********              gs_documentheader-doc_date = lv_bldat AND
**********              gs_documentheader-pstng_date = lv_budat AND
**********              gs_documentheader-doc_type = gs_upload-blart AND
**********              gs_documentheader-fisc_year = gs_upload-budat+4(4) AND
**********              gs_documentheader-fis_period = gs_upload-monat AND
*************      gs_documentheader-username = sy-uname.
*************      gs_documentheader-header_txt = gs_upload-bktxt.
*************    gs_documentheader-reason_rev = gs_upload-stgrd.
*************  gs_documentheader-acc_principle = gs_upload-ledger_group.
*************  gs_documentheader-obj_key_inv = gs_upload-rebzg.
*********              gs_documentheader-ref_doc_no = gs_upload-xblnr AND
             gv_file_counter = gs_upload-row_number ).
***********      change in header control fields
      PERFORM frm_post_document.
      CLEAR lv_line_cnt.
      PERFORM frm_check_control_values_hdr.
      PERFORM frm_build_document_header.
      CLEAR gt_upload_store .
    ENDIF.

    APPEND gs_upload TO gt_upload_store . "store the processed lines for rpt

    gv_file_counter = gs_upload-row_number.
*
    ADD 1 TO lv_line_cnt.
    ADD 1 TO gv_posnr.
* DO Checks
*****    PERFORM frm_get_koart.
    PERFORM frm_check_control_values_item.
    PERFORM frm_check_ledger_group.
    PERFORM frm_check_amounts.

    PERFORM frm_check_accruals.
*    *
    CASE gs_tbsl-koart.
      WHEN gc_koart_gl.   .               "S
        PERFORM frm_process_gl_row.
      WHEN gc_koart_vendor.               "K
        PERFORM frm_process_vendor_row.
      WHEN gc_koart_customer.             "D
        PERFORM frm_process_customer_row.
      WHEN gc_koart_asset.                "A
        PERFORM frm_process_asset_row.
      WHEN OTHERS.
        MESSAGE e204(rf) WITH gs_tbsl-koart INTO gv_dummy..
* Account type '$' is not supported
        PERFORM frm_add_error_to_log.
        gv_do_not_process = gc_x.
    ENDCASE.
*    IF sy-tcode EQ gc-tcode_fb01.
    PERFORM frm_build_currency_data.  "for this line
*    ELSE.
*      PERFORM frm_build_currency_data_fbb1.
*    ENDIF.
*
    IF  lv_line_cnt > gv_doc_max_line_cnt.
      PERFORM frm_build_balancing_entry.
***************      build a gl line and currency line
**      data gv_currencyamount-amt_doccur type BAPIDOCCUR.
      PERFORM frm_post_document.
****
****      PERFORM frm_check_control_values_hdr.
      PERFORM frm_build_document_header.
****
      PERFORM frm_build_balancing_entry.
      CLEAR lv_line_cnt.
    ENDIF.

  ENDLOOP.
*
  PERFORM frm_post_document.
******  PERFORM frm_display_results .
*
ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_display_results
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_display_results .

  DATA ls_layout  TYPE  slis_layout_alv.

  DATA ls_variant TYPE  disvariant.
  ls_variant-report = sy-repid.
  ls_variant-handle = 'RRES'.
*  ls_variant-log_group =
*  ls_variant-username =
*  ls_variant-variant =
*  ls_variant-text =
*  ls_variant-dependvars = ' '.

***remove all 'S' messages when errors have occured
  DATA lt_report TYPE TABLE OF zcfis_journal_upload_np.
  DATA ls_report LIKE LINE OF gt_report.
  lt_report = gt_report.
  SORT lt_report BY row_number type.
  DELETE ADJACENT DUPLICATES FROM lt_report COMPARING row_number type.
  LOOP AT lt_report INTO ls_report WHERE type = 'E'.
    DELETE gt_report WHERE row_number = ls_report-row_number AND type = 'S'.
  ENDLOOP.
  CLEAR lt_report.

***  DATA lv_sy_tabix TYPE sy-tabix.
***  LOOP AT gt_report INTO gs_report.
  DELETE gt_report WHERE id = 'RW' AND number = '633'.
***    lv_sy_tabix = sy-tabix.
***    IF gs_report-id = 'RW' AND gs_report-number = '633'.
***      REPLACE ALL OCCURRENCES OF 'YYYYMMDD' IN gs_report-message WITH 'MMDDYYYY'.
***      REPLACE ALL OCCURRENCES OF '(year, month, day)' IN gs_report-message WITH '( month, day, year )'.
***    ENDIF.
***    MODIFY gt_report FROM gs_report INDEX lv_sy_tabix.
***  ENDLOOP.

***  REPLACE  ALL OCCURRENCES OF   'YYYYMMDD'
***          IN  TABLE gt_report WITH 'MMDDYYY'  IN CHARACTER MODE
***                    RESPECTING CASE.



  CLEAR ls_layout.
  ls_layout-zebra = gc_x.

  PERFORM frm_build_fieldcat.

  PERFORM frm_alv_eventtab_build USING gt_alv_events[].
  "PERFORM f_alv_sort_build.
  PERFORM frm_alv_comment_build_top  USING gt_alv_list_top_of_page[].

  CALL FUNCTION 'REUSE_ALV_GRID_DISPLAY'
    EXPORTING
*     I_INTERFACE_CHECK  = ' '
*     I_BYPASSING_BUFFER = ' '
*     I_BUFFER_ACTIVE    = ' '
      i_callback_program = sy-repid
*     I_CALLBACK_PF_STATUS_SET          = ' '
*     I_CALLBACK_USER_COMMAND           = ' '
*     I_CALLBACK_TOP_OF_PAGE            = ' '
*     I_CALLBACK_HTML_TOP_OF_PAGE       = ' '
*     I_CALLBACK_HTML_END_OF_LIST       = ' '
*     I_STRUCTURE_NAME   = I_STRUCTURE_NAME
*     I_BACKGROUND_ID    = ' '
*     I_GRID_TITLE       = I_GRID_TITLE
*     I_GRID_SETTINGS    = I_GRID_SETTINGS
*     IS_LAYOUT          = IS_LAYOUT
      it_fieldcat        = gt_fieldcat
*     IT_EXCLUDING       = IT_EXCLUDING
*     IT_SPECIAL_GROUPS  = IT_SPECIAL_GROUPS
*     IT_SORT            = IT_SORT
*     IT_FILTER          = IT_FILTER
*     IS_SEL_HIDE        = IS_SEL_HIDE
*     I_DEFAULT          = 'X'
      i_save             = 'A'
      is_variant         = ls_variant
      it_events          = gt_alv_events[]
*     IT_EVENT_EXIT      = IT_EVENT_EXIT
*     IS_PRINT           = IS_PRINT
*     IS_REPREP_ID       = IS_REPREP_ID
*     I_SCREEN_START_COLUMN             = 0
*     I_SCREEN_START_LINE               = 0
*     I_SCREEN_END_COLUMN               = 0
*     I_SCREEN_END_LINE  = 0
*     I_HTML_HEIGHT_TOP  = 0
*     I_HTML_HEIGHT_END  = 0
*     IT_ALV_GRAPHICS    = IT_ALV_GRAPHICS
*     IT_HYPERLINK       = IT_HYPERLINK
*     IT_ADD_FIELDCAT    = IT_ADD_FIELDCAT
*     IT_EXCEPT_QINFO    = IT_EXCEPT_QINFO
*     IR_SALV_FULLSCREEN_ADAPTER        = IR_SALV_FULLSCREEN_ADAPTER
*   IMPORTING
*     E_EXIT_CAUSED_BY_CALLER           = E_EXIT_CAUSED_BY_CALLER
*     ES_EXIT_CAUSED_BY_USER            = ES_EXIT_CAUSED_BY_USER
    TABLES
      t_outtab           = gt_report
    EXCEPTIONS
      program_error      = 1
      OTHERS             = 2.
  IF sy-subrc <> 0.
* Implement suitable error handling here
  ENDIF.


ENDFORM.


FORM frm_build_fieldcat.

*DATA I_PROGRAM_NAME         TYPE SY-REPID.
*DATA I_INTERNAL_TABNAME     TYPE SLIS_TABNAME.
*DATA I_STRUCTURE_NAME       TYPE DD02L-TABNAME.
*DATA I_CLIENT_NEVER_DISPLAY TYPE SLIS_CHAR_1.
*DATA I_INCLNAME             TYPE TRDIR-NAME.
*DATA I_BYPASSING_BUFFER     TYPE CHAR01.
*DATA I_BUFFER_ACTIVE        TYPE CHAR01.
*  DATA gt_fieldcat            TYPE slis_t_fieldcat_alv.

  DATA ls_fieldcat LIKE LINE OF gt_fieldcat.
  CLEAR gt_fieldcat.

  CALL FUNCTION 'REUSE_ALV_FIELDCATALOG_MERGE'
    EXPORTING
*     I_PROGRAM_NAME         = I_PROGRAM_NAME
*     I_INTERNAL_TABNAME     = I_INTERNAL_TABNAME
      i_structure_name       = 'ZCFIS_JOURNAL_UPLOAD_NP'
*     I_CLIENT_NEVER_DISPLAY = 'X'
*     I_INCLNAME             = I_INCLNAME
*     I_BYPASSING_BUFFER     = I_BYPASSING_BUFFER
*     I_BUFFER_ACTIVE        = I_BUFFER_ACTIVE
    CHANGING
      ct_fieldcat            = gt_fieldcat
    EXCEPTIONS
      inconsistent_interface = 1
      program_error          = 2
      OTHERS                 = 3.
  IF sy-subrc <> 0.
* Implement suitable error handling here
  ENDIF.


  LOOP AT gt_fieldcat INTO ls_fieldcat.

    CASE ls_fieldcat-fieldname.
****      WHEN 'PROJN'.
****        ls_fieldcat-seltext_l = 'WBS'.
****        ls_fieldcat-seltext_m = 'WBS'.
****        ls_fieldcat-seltext_s = 'WBS'.
****        ls_fieldcat-reptext_ddic = 'WBS'.
      WHEN 'WRBTR' .
        ls_fieldcat-seltext_l = 'Absolute amount'.
        ls_fieldcat-seltext_m = 'Absolute amount'.
        ls_fieldcat-seltext_s = 'Absolute amount'.
        ls_fieldcat-reptext_ddic = 'Absolute amount'.
      WHEN 'TRADING_PARTNER'.
        ls_fieldcat-seltext_l = 'Trading partner'.
        ls_fieldcat-seltext_m = 'Trading partner'.
        ls_fieldcat-seltext_s = 'Trading partner'.
        ls_fieldcat-reptext_ddic = 'Trading partner'.
      WHEN 'CONSOL_TRAN_TYPE'.
        ls_fieldcat-seltext_l = 'Transaction Type'.
        ls_fieldcat-seltext_m = 'Tran. Type'.
        ls_fieldcat-seltext_s = 'Tran. Type'.
        ls_fieldcat-reptext_ddic = 'Tran. Type'.
      WHEN 'ALT_RECON_ACCNT'.
        ls_fieldcat-seltext_l = 'Alt. Recon. Account'.
        ls_fieldcat-seltext_m = 'Alt. Recon. Acc'.
        ls_fieldcat-seltext_s = 'Alt. Recon. Acc'.
        ls_fieldcat-reptext_ddic = 'Alt. Recon. Acc'.
      WHEN 'DMBTR'.
        ls_fieldcat-seltext_l = 'Absolute Amt (LC) (CDF)'.
        ls_fieldcat-seltext_m = 'Absolute Amt (LC) (CDF)'.
        ls_fieldcat-seltext_s = 'Absolute Amt (LC) (CDF)'.
        ls_fieldcat-reptext_ddic = 'Absolute Amt (LC) (CDF)'.
      WHEN 'DMBE2'.
        ls_fieldcat-seltext_l = 'Absolute Amt (GC) (USD)'.
        ls_fieldcat-seltext_m = 'Absolute Amt (GC) (USD)'.
        ls_fieldcat-seltext_s = 'Absolute Amt (GC) (USD)'.
        ls_fieldcat-reptext_ddic = 'Absolute Amt (GC) (USD)'.
      WHEN 'DMBE3'.
        ls_fieldcat-seltext_l = 'Absolute Amt (GC2) (CDF)'.
        ls_fieldcat-seltext_m = 'Absolute Amt (GC2) (CDF)'.
        ls_fieldcat-seltext_s = 'Absolute Amt (GC2) (CDF)'.
        ls_fieldcat-reptext_ddic = 'Absolute Amt (GC2) (CDF)'.
      WHEN 'SP_ASSET_TRAN_TYPE'.
        ls_fieldcat-seltext_l = 'Special Asset Trans Type'.
        ls_fieldcat-seltext_m = 'Special Asset Trans Type'.
        ls_fieldcat-seltext_s = 'Special Asset Trans Type'.
        ls_fieldcat-reptext_ddic = 'Special Asset Trans Type'.
      WHEN 'BLDAT'.
        ls_fieldcat-seltext_l = 'Document Date'.
        ls_fieldcat-seltext_m = 'Document Date'.
        ls_fieldcat-seltext_s = 'Document Date'.
        ls_fieldcat-reptext_ddic = 'Document Date'.
      WHEN 'BUDAT'.
        ls_fieldcat-seltext_l = 'Posting Date'.
        ls_fieldcat-seltext_m = 'Posting Date'.
        ls_fieldcat-seltext_s = 'Posting Date'.
        ls_fieldcat-reptext_ddic = 'Posting Date'.
      WHEN 'STODT'.
        ls_fieldcat-seltext_l = 'Reversal Date'.
        ls_fieldcat-seltext_m = 'Reversal Date'.
        ls_fieldcat-seltext_s = 'Reversal Date'.
        ls_fieldcat-reptext_ddic = 'Reversal Date'.
      WHEN 'ZFBDT'.
        ls_fieldcat-seltext_l = 'Baseline Date'.
        ls_fieldcat-seltext_m = 'Baseline Date'.
        ls_fieldcat-seltext_s = 'Baseline Date'.
        ls_fieldcat-reptext_ddic = 'Baseline Date'.
      WHEN 'MENGE'.
        ls_fieldcat-seltext_l = 'Quantity'.
        ls_fieldcat-seltext_m = 'Quantity'.
        ls_fieldcat-seltext_s = 'Quantity'.
        ls_fieldcat-reptext_ddic = 'Quantity'.
      WHEN OTHERS.
    ENDCASE.

    MODIFY gt_fieldcat FROM  ls_fieldcat INDEX sy-tabix.

  ENDLOOP.


ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_check_control_values_hdr
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_check_control_values_hdr .

*  check the company code
  CLEAR gs_t001.
  SELECT SINGLE bukrs, waers
    FROM t001
    INTO ( @gs_t001-bukrs, @gs_t001-waers )
  WHERE bukrs = @gs_upload-bukrs.
  IF sy-subrc NE 0.
    MESSAGE e011(zfi_msg) WITH gs_upload-bukrs gs_upload-row_number INTO gv_dummy.
* Company code & on  row & is not valid
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.

  ENDIF.
*
ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_check_control_values_item
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_check_control_values_item .

  DATA ls_t042e TYPE t042e.

****  check the posting key
  CLEAR gs_tbsl.
  READ TABLE gt_tbsl INTO gs_tbsl WITH KEY bschl = gs_upload-bschl.
  IF sy-subrc NE 0.
    MESSAGE e010(zfi_msg) WITH gs_upload-bschl gs_upload-row_number INTO gv_dummy.
* Posting key & on  row & is not valid
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.
****
*check ACCOUNTING_PRINCIPLE
  CLEAR gs_tacc_trgt_ldgr.
  IF NOT gs_upload-ledger_group IS INITIAL.
    SELECT SINGLE *
      FROM tacc_trgt_ldgr
      INTO @gs_tacc_trgt_ldgr
    WHERE acc_principle =  @gs_upload-ledger_group.
    IF sy-subrc NE 0.
      MESSAGE e009(zfi_msg) WITH gs_upload-ledger_group gs_upload-row_number INTO gv_dummy.
* Ledger Group & on  row & is not valid
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
  ENDIF.
*
  IF  NOT ( gs_t001 IS INITIAL AND gs_tacc_trgt_ldgr IS INITIAL ).
*  Check ledger group company code authorisation
    AUTHORITY-CHECK OBJECT 'F_FAGL_LDR'
     ID 'BUKRS' FIELD gs_t001-bukrs
     ID 'GLRLDNR' FIELD gs_tacc_trgt_ldgr-ldgrp_gl
     ID 'GLRRCTY' FIELD '*'
     ID 'GLRVERS' FIELD '*'
     ID 'ACTVT' FIELD '01'.
    IF sy-subrc <> 0.
      MESSAGE e012(zfi_msg) WITH gs_tacc_trgt_ldgr-ldgrp_gl INTO gv_dummy.
* No authorization for posting to Ledger Group &
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
  ENDIF.
*
***************  IF gs_upload-waers = gs_t001-waers.
****************   docuemnt currency =  local currency
***************    IF gs_upload-dmbtr IS NOT INITIAL.
***************      MESSAGE e015(zfi_msg) INTO gv_dummy.
**************** Do not fill Local currency amount where Trans Curr Key = Local Curr Key
***************      PERFORM frm_add_error_to_log.
***************      gv_do_not_process = gc-x.
***************    ENDIF.
***************  ENDIF.
*
  CASE gs_tbsl-koart.
    WHEN gc_koart_vendor OR  gc_koart_customer.                "K   "D
      IF NOT gs_upload-ledger_group IS INITIAL.
        MESSAGE e013(zfi_msg) WITH 'not allowed' 'Customer / Vendor' INTO gv_dummy..
* Ledger Group & for & postings
        PERFORM frm_add_error_to_log.
        gv_do_not_process = gc_x.
      ENDIF.
    WHEN gc_koart_gl OR gc_koart_asset.                        "S  "A
****      IF gs_upload-ledger_group IS INITIAL.
****        MESSAGE e013(zfi_msg) WITH 'required' 'GL / Asset'  INTO gv_dummy.
***** Ledger Group & for & postings
****        PERFORM frm_add_error_to_log.
****        gv_do_not_process = gc-x.
****      ENDIF.
  ENDCASE.
*
  IF NOT gs_upload-zlsch IS INITIAL.
    CASE gs_tbsl-koart.
      WHEN gc_koart_vendor OR  gc_koart_customer.
        CLEAR ls_t042e.
        SELECT SINGLE zbukr
          FROM t042e
          INTO @ls_t042e-zbukr
          WHERE zbukr = @gs_upload-bukrs
        AND zlsch = @gs_upload-zlsch.
        IF sy-subrc NE 0.
          MESSAGE e018(zfi_msg) WITH  gs_upload-bukrs gs_upload-zlsch '&3' gs_upload-row_number INTO gv_dummy .
* Payment method &, is not valid for Company Code &, on row &
          PERFORM frm_add_error_to_log.
          gv_do_not_process = gc_x.
        ENDIF.

      WHEN OTHERS.
        MESSAGE e017(zfi_msg) INTO gv_dummy.
* Payment method can only be entered on postings to customer/vendor account
        PERFORM frm_add_error_to_log.
        gv_do_not_process = gc_x.
    ENDCASE.
  ENDIF.
*
  IF ( gs_upload-ebeln IS NOT INITIAL AND gs_upload-ebelp IS INITIAL )   OR
     ( gs_upload-ebeln IS INITIAL AND gs_upload-ebelp IS NOT INITIAL ).
    MESSAGE e019(zfi_msg) WITH gs_upload-row_number INTO gv_dummy.
* Both Purchase Order an PO Item are required on row &
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.
*
ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_add_error_to_log
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_add_error_to_log.

**gs_upload-row_number
  CLEAR gs_return.
  MOVE syst-msgid TO gs_return-id.
  MOVE syst-msgty TO gs_return-type.
  MOVE syst-msgno TO gs_return-number.
  MOVE syst-msgv1 TO gs_return-message_v1.
  MOVE syst-msgv2 TO gs_return-message_v2.
  MOVE syst-msgv3 TO gs_return-message_v3.
  MOVE syst-msgv4 TO gs_return-message_v4.
  MESSAGE ID sy-msgid TYPE sy-msgty NUMBER sy-msgno
            INTO gs_return-message
            WITH sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4.
*
  CASE gs_return-type.
    WHEN 'E'.
      gs_report-icon = '@5C@'. "red icon
    WHEN 'S'.
      gs_report-icon = '@5B@'. "green icon
    WHEN OTHERS.
      gs_report-icon = '@09@'. "YELLOW icon
  ENDCASE.
*
  DATA lv_ps_project_id TYPE ps_pspid.
  DATA ls_return           TYPE bapiret2.

  IF NOT gs_upload-projn IS INITIAL.

    lv_ps_project_id = gs_upload-projn.
    CALL FUNCTION 'RPM_CHECK_PSPID_MASK'
      EXPORTING
        iv_ps_project_id = lv_ps_project_id
      IMPORTING
        return           = ls_return
*     TABLES
*       PROJECT_LIST     = PROJECT_LIST
      .
    IF ls_return-type EQ 'E'.
      CLEAR gs_upload-projn.

    ENDIF.
  ENDIF.

  MOVE gs_upload TO gs_report-infile.
  MOVE gs_return TO gs_report-bapiret.
*
  APPEND gs_report TO gt_report.

ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_process_gl_row
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_process_gl_row.
*
  DATA lv_posid(24).
  DATA lv_prps_posid TYPE prps-posid.
*
  AUTHORITY-CHECK OBJECT 'F_BKPF_BUK'
        ID 'BUKRS' FIELD gs_t001-bukrs
        ID 'ACTVT' FIELD '01'.

  IF sy-subrc <> 0.
    MESSAGE e607(9p) WITH 'GL'    INTO gv_dummy.
* No authorization for account type &1
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.
*
  DATA lv_date(8) TYPE n.
  CLEAR gs_accountgl.
*  CLEAR gs_accountgl_accr.
*  gs_ACCOUNTGL-

  gs_accountgl-itemno_acc = gv_posnr.
  IF gs_tbsl-koart = 'S'.
    SHIFT gs_upload-hkont RIGHT DELETING TRAILING ' '.
    OVERLAY gs_upload-hkont WITH '0000000000'.
    gs_accountgl-gl_account = gs_upload-hkont.
  ENDIF.
*
  gs_accountgl-item_text = gs_upload-sgtxt.
  gs_accountgl-acct_type = gs_tbsl-koart.
  gs_accountgl-doc_type = gs_upload-blart.
  gs_accountgl-comp_code = gs_upload-bukrs.
  gs_accountgl-fis_period = gs_upload-monat.
  gs_accountgl-pstng_date = gs_upload-budat.
  lv_date = gs_upload-budat.
  gs_accountgl-pstng_date = lv_date+4(4) && lv_date(4).
  gs_accountgl-fisc_year =  gs_accountgl-pstng_date(4).
  gs_accountgl-alloc_nmbr = gs_upload-zuonr.
  gs_accountgl-tax_code = gs_upload-mwskz.
  gs_accountgl-costcenter = gs_upload-kostl.
  gs_accountgl-profit_ctr = gs_upload-prctr.

  IF NOT gs_upload-projn IS INITIAL.             "wbs element

    DATA lv_ps_project_id TYPE ps_pspid.
    DATA ls_return           TYPE bapiret2.


    lv_ps_project_id = gs_upload-projn.
    CALL FUNCTION 'RPM_CHECK_PSPID_MASK'
      EXPORTING
        iv_ps_project_id = lv_ps_project_id
      IMPORTING
        return           = ls_return
*     TABLES
*       PROJECT_LIST     = PROJECT_LIST
      .
    IF ls_return-type EQ 'E'.
      CLEAR gs_accountgl-wbs_element.
      CLEAR gs_upload-projn.    " if not intialized causes a runtime error
      IF ls_return-id = 'CJ' AND ls_return-number = '609'.
        MESSAGE e059(zfi_msg) WITH ls_return-message_v1 INTO gv_dummy.
* WBS Key does not correspond to mask: &
      ELSE.
        MESSAGE ID ls_return-id TYPE ls_return-type NUMBER ls_return-number
        WITH ls_return-message_v1 ls_return-message_v2 ls_return-message_v3 ls_return-message_v4 INTO gv_dummy.
      ENDIF.
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ELSE.
      gs_accountgl-wbs_element =  gs_upload-projn.
    ENDIF.
  ENDIF.
  gs_accountgl-orderid = gs_upload-aufnr.
  IF gs_tbsl-koart = 'A'.
    gs_accountgl-asset_no = gs_upload-hkont.   " ????
  ENDIF.

  gs_accountgl-de_cre_ind = gs_tbsl-shkzg.

  gs_accountgl-quantity  = gs_upload-menge.
  gs_accountgl-base_uom  = gs_upload-meins.
  gs_accountgl-trade_id = gs_upload-trading_partner.
  gs_accountgl-po_number = gs_upload-ebeln.
  gs_accountgl-po_item = gs_upload-ebelp.

  gs_accountgl-cs_trans_t = gs_upload-consol_tran_type.
  gs_accountgl-housebankid = gs_upload-hbkid.
  gs_accountgl-housebankacctid = gs_upload-hktid.

*&–>..Begin of Modification C-002
  " add material
  gs_accountgl-material = |{ gs_upload-matnr ALPHA = IN WIDTH = 18 }|.
*&<–..End of Modification C-002

*
  IF   gv_accrual NE gc_x.
    APPEND gs_accountgl TO gt_accountgl.
***BEGIN OF INSERTION CH-1321
    gs_acc_ext-posnr = gv_posnr.
    gs_acc_ext-xnegp = gs_upload-xnegp.
    APPEND VALUE #( structure  = 'ZCFIS_ACC_EXT4'
                    valuepart1 = gs_acc_ext ) TO gt_extension2.
***END OF INSERTION CH-1321
  ELSE.  "accrual
*    MOVE-CORRESPONDING gs_accountgl TO gs_accountgl_accr.
*    APPEND gs_accountgl_accr TO gt_accountgl_accr.
  ENDIF.


ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_process_vendor_row
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_process_vendor_row.

  AUTHORITY-CHECK OBJECT 'F_BKPF_BEK'
   ID 'BRGRU' FIELD '*'
   ID 'ACTVT' FIELD '01'.

  IF sy-subrc <> 0.
    MESSAGE e607(9p) WITH 'Vendor'    INTO gv_dummy.
* No authorization for account type &1
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.

  DATA lv_date(8) TYPE n.
  CLEAR gs_accountpayable.

  gs_accountpayable-itemno_acc = gv_posnr.

  gs_accountpayable-vendor_no = gs_upload-hkont.     "lifnr.  "??????
  IF gs_accountpayable-vendor_no CO ' 1234567890'.
    SHIFT gs_accountpayable-vendor_no RIGHT DELETING TRAILING ' '.
    OVERLAY gs_accountpayable-vendor_no WITH '0000000000'.
  ENDIF.

*  gs_accountpayable-gl_account = gs_upload-hkont.
  gs_accountpayable-comp_code = gs_upload-bukrs.
  gs_accountpayable-pmnttrms = gs_upload-zterm.
**  gs_accountpayable-bline_date = gs_upload-zfbdt.
  lv_date = gs_upload-zfbdt.
** lv_date = gs_accountpayable-bline_date .
  gs_accountpayable-bline_date =  lv_date+4(4) && lv_date(4).
  gs_accountpayable-alloc_nmbr = gs_upload-zuonr.
  gs_accountpayable-item_text  = gs_upload-sgtxt.
  gs_accountpayable-sp_gl_ind  = gs_upload-umskz.
  gs_accountpayable-partner_bk = gs_upload-bvtyp.
  gs_accountpayable-bank_id  = gs_upload-hbkid.
  gs_accountpayable-housebankacctid = gs_upload-hktid.

  APPEND gs_accountpayable TO gt_accountpayable.
***BEGIN OF INSERTION CH-1321
  gs_acc_ext-posnr = gv_posnr.
  gs_acc_ext-xnegp = gs_upload-xnegp.
  APPEND VALUE #( structure  = 'ZCFIS_ACC_EXT4'
                  valuepart1 = gs_acc_ext ) TO gt_extension2.
***END OF INSERTION CH-1321

ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_process_customer_row
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_process_customer_row.
*
  DATA lv_date(8) TYPE n.
*
  CLEAR gs_accountreceivable.
  AUTHORITY-CHECK OBJECT 'F_BKPF_BED'
   ID 'BRGRU' FIELD '*'
   ID 'ACTVT' FIELD '01'.

  IF sy-subrc <> 0.
    MESSAGE e607(9p) WITH 'Customer'    INTO gv_dummy.
* No authorization for account type &1
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.
*
  gs_accountreceivable-itemno_acc = gv_posnr.

  gs_accountreceivable-customer  = gs_upload-hkont.   "should have customer number
  SHIFT gs_accountreceivable-customer RIGHT DELETING TRAILING ' '.
  OVERLAY gs_accountreceivable-customer WITH '0000000000'.
*****  gs_accountreceivable-GL_ACCOUNT = gs_upload-     ????????????
  gs_accountreceivable-comp_code = gs_upload-bukrs.

  gs_accountreceivable-pmnttrms = gs_upload-zterm.
*****  gs_accountreceivable-bline_date = gs_upload-zfbdt.
  lv_date = gs_upload-zfbdt.
** lv_date = gs_accountpayable-bline_date .
  gs_accountreceivable-bline_date =  lv_date+4(4) && lv_date(4).
  gs_accountreceivable-alloc_nmbr = gs_upload-zuonr.
  gs_accountreceivable-item_text = gs_upload-sgtxt.
  gs_accountreceivable-partner_bk = gs_upload-bvtyp.

  gs_accountreceivable-bank_id  = gs_upload-hbkid.
  gs_accountreceivable-sp_gl_ind = gs_upload-umskz.
  gs_accountreceivable-profit_ctr = gs_upload-prctr.
  gs_accountreceivable-housebankacctid = gs_upload-hktid.

  APPEND gs_accountreceivable TO gt_accountreceivable.
***BEGIN OF INSERTION CH-1321
  gs_acc_ext-posnr = gv_posnr.
  gs_acc_ext-xnegp = gs_upload-xnegp.
  APPEND VALUE #( structure  = 'ZCFIS_ACC_EXT4'
                  valuepart1 = gs_acc_ext ) TO gt_extension2.
***END OF INSERTION CH-1321
ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_process_asset_row
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_process_asset_row.
*
  AUTHORITY-CHECK OBJECT 'A_B_ANLKL'
   ID 'ANLKL' FIELD '*'
   ID 'BUKRS' FIELD gs_t001-bukrs
   ID 'ACTVT' FIELD '01'.

  IF sy-subrc <> 0.
    MESSAGE e607(9p) WITH 'Asset'    INTO gv_dummy.
* No authorization for account type &1
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.
*
ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_post_document
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_post_document.
*
  DATA lv_obj_type  LIKE  bapiache09-obj_type.
  DATA lv_obj_key LIKE  bapiache09-obj_key.
  DATA lv_obj_sys LIKE  bapiache09-obj_sys.


**  BREAK mmakda.   "POSTING
  IF  NOT gs_documentheader IS INITIAL.
    IF gv_do_not_process EQ gc_x OR gv_test = gc_x.
**check
      CALL FUNCTION 'BAPI_ACC_DOCUMENT_CHECK'
        EXPORTING
          documentheader    = gs_documentheader
*         CUSTOMERCPD       = CUSTOMERCPD
*         CONTRACTHEADER    = CONTRACTHEADER
        TABLES
          accountgl         = gt_accountgl
          accountreceivable = gt_accountreceivable
          accountpayable    = gt_accountpayable
*         ACCOUNTTAX        = ACCOUNTTAX
          currencyamount    = gt_currencyamount
*         CRITERIA          = CRITERIA
*         VALUEFIELD        = VALUEFIELD
          extension1        = gt_extension1
          return            = gt_return
*         PAYMENTCARD       = PAYMENTCARD
*         CONTRACTITEM      = CONTRACTITEM
          extension2        = gt_extension2 "CH-1321 +
*         REALESTATE        = REALESTATE
*         ACCOUNTWT         = ACCOUNTWT
        .

    ELSEIF gv_do_not_process IS INITIAL AND gv_test IS INITIAL.
***post
      CALL FUNCTION 'BAPI_ACC_DOCUMENT_POST'
        EXPORTING
          documentheader    = gs_documentheader
*         CUSTOMERCPD       = CUSTOMERCPD
*         CONTRACTHEADER    = CONTRACTHEADER
* IMPORTING
*         OBJ_TYPE          = OBJ_TYPE
*         OBJ_KEY           = OBJ_KEY
*         OBJ_SYS           = OBJ_SYS
        TABLES
          accountgl         = gt_accountgl
          accountreceivable = gt_accountreceivable
          accountpayable    = gt_accountpayable
*         ACCOUNTTAX        = ACCOUNTTAX
          currencyamount    = gt_currencyamount
*         CRITERIA          = CRITERIA
*         VALUEFIELD        = VALUEFIELD
          extension1        = gt_extension1
          return            = gt_return
*         PAYMENTCARD       = PAYMENTCARD
*         CONTRACTITEM      = CONTRACTITEM
          extension2        = gt_extension2 "CH-1321 +
*         REALESTATE        = REALESTATE
*         ACCOUNTWT         = ACCOUNTWT
        .


      LOOP AT gt_return INTO gs_return WHERE type CA 'AE'.
      ENDLOOP.
      IF sy-subrc <> 0.
        CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'
          EXPORTING
            wait = gc_x.
      ENDIF.

    ENDIF.
  ENDIF.

*
**  move bapi return to report
  LOOP AT gt_return INTO gs_return.
    LOOP AT gt_upload_store INTO  gs_upload_store WHERE  row_number = gv_file_counter.
      MOVE-CORRESPONDING gs_upload_store TO gs_report-infile.
      MOVE gs_return TO gs_report-bapiret.
      gs_report-row_number = gv_file_counter.
*
      CASE gs_return-type.
        WHEN 'E'.
          gs_report-icon = '@5C@'. "red icon
        WHEN 'S'.
          gs_report-icon = '@5B@'. "green icon
        WHEN OTHERS.
          gs_report-icon = '@09@'. "YELLOW icon
      ENDCASE.
*
      IF NOT gs_report-projn IS INITIAL.             "wbs element

        DATA lv_ps_project_id TYPE ps_pspid.
        DATA ls_return           TYPE bapiret2.

***    IF gs_upload-projn NA '-'.
***      CALL FUNCTION 'CONVERSION_EXIT_ABPSN_OUTPUT'
***        EXPORTING
***          input  = gs_upload-projn
***        IMPORTING
***          output = lv_ps_project_id.
***    ELSE.
***      lv_ps_project_id = gs_upload-projn.
***    ENDIF.

        lv_ps_project_id = gs_report-projn.
        CALL FUNCTION 'RPM_CHECK_PSPID_MASK'
          EXPORTING
            iv_ps_project_id = lv_ps_project_id
          IMPORTING
            return           = ls_return
*     TABLES
*           PROJECT_LIST     = PROJECT_LIST
          .
        IF ls_return-type EQ 'E'.
          CLEAR gs_accountgl-wbs_element.
          CLEAR gs_report-projn.    " if not initialized causes runtime error
        ENDIF.
      ENDIF.
*
      APPEND gs_report TO gt_report.
    ENDLOOP.
  ENDLOOP.

ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_check_ledger_group
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_check_ledger_group.
*
  CASE gs_tbsl-koart.
    WHEN gc_koart_vendor OR  gc_koart_customer.               "K  "D
      IF NOT gs_upload-ledger_group IS INITIAL.
        MESSAGE e013(zfi_msg) WITH 'not allowed' 'Customer / Vendor' INTO gv_dummy.
* Ledger Group & for & postings
        PERFORM frm_add_error_to_log.
        gv_do_not_process = gc_x.
      ENDIF.
    WHEN gc_koart_gl OR gc_koart_asset.              "S "A
***      IF gs_upload-ledger_group IS INITIAL.
***        MESSAGE e013(zfi_msg) WITH 'required' 'GL / Asset'  INTO gv_dummy.
**** Ledger Group & for & postings
***        PERFORM frm_add_error_to_log.
***        gv_do_not_process = gc-x.
***      ENDIF.
  ENDCASE.
*
ENDFORM.
*&---------------------------------------------------------------------*
*& FORM FORM frm_check_amounts
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_check_amounts.
*
  DATA lv_decimals TYPE i.
  DATA ls_finsc_ld_cmp TYPE finsc_ld_cmp.
*
  IF gs_upload-dmbe3  IS NOT INITIAL.   "
**get the curtyp from FINSC_LD_CMP
    SELECT SINGLE *
      FROM finsc_ld_cmp
      INTO ls_finsc_ld_cmp
      WHERE rldnr = '0L'    "use constant as per FDS
    AND  bukrs = gs_upload-bukrs.

    IF sy-subrc NE 0.
      MESSAGE e038(zfi_msg) WITH gs_upload-bukrs INTO gv_dummy.
* Do not enter GC2 amount for Company Code &
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
  ENDIF.

  PERFORM frm_determine_amount_decimals USING  gs_upload-wrbtr CHANGING lv_decimals.
  IF lv_decimals > 2.
    MESSAGE e014(zfi_msg) WITH ' ' gs_upload-row_number gs_upload-wrbtr  INTO gv_dummy.
* Amount & on row &, not formatted correctly - &
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.
  PERFORM frm_determine_amount_decimals USING  gs_upload-dmbtr CHANGING lv_decimals.
  IF lv_decimals > 2.
    MESSAGE e014(zfi_msg) WITH ' ' gs_upload-row_number gs_upload-wrbtr  INTO gv_dummy.
* Amount & on row &, not formatted correctly - &
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.
  PERFORM frm_determine_amount_decimals USING  gs_upload-dmbe2 CHANGING lv_decimals.
  IF lv_decimals > 2.
    MESSAGE e014(zfi_msg) WITH ' ' gs_upload-row_number gs_upload-wrbtr  INTO gv_dummy.
* Amount & on row &, not formatted correctly - &
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.
  PERFORM frm_determine_amount_decimals USING  gs_upload-dmbe3 CHANGING lv_decimals.
  IF lv_decimals > 2.
    MESSAGE e014(zfi_msg) WITH ' ' gs_upload-row_number gs_upload-wrbtr  INTO gv_dummy.
* Amount & on row &, not formatted correctly - &
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.


  IF gs_t001-waers = gs_upload-waers AND gs_upload-wrbtr NE gs_upload-dmbtr AND gs_upload-dmbtr IS NOT INITIAL.
***      BREAK mmakda.   " amount validation
    MESSAGE e058(zfi_msg) WITH gs_upload-waers  INTO gv_dummy.
* LC amount must equal TC amount for TC &
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ELSEIF gs_t001-waers = 'USD'  AND gs_upload-dmbtr NE gs_upload-dmbe2 AND gs_upload-dmbe2 IS NOT INITIAL.
**      BREAK mmakda.   " amount validation
***        Error     "LC amount must equal GC amount for LC xxx (T001-WAERS)"
    IF NOT gs_upload-dmbe2 IS INITIAL.
      MESSAGE e057(zfi_msg) WITH gs_t001-waers INTO gv_dummy.
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
  ELSEIF gs_t001-waers NE 'USD'    AND  gs_upload-waers = 'USD'   AND   gs_upload-wrbtr NE gs_upload-dmbe2 AND gs_upload-dmbe2 IS NOT INITIAL.
    MESSAGE e060(zfi_msg) WITH gs_upload-waers INTO gv_dummy.
* GC amount must equal TC amount for LC
    PERFORM frm_add_error_to_log.
    gv_do_not_process = gc_x.
  ENDIF.

ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_determine_amount_decimals USING pv_amount TYPE char20 CHANGING cv_decimals TYPE i.
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_determine_amount_decimals USING pv_amount TYPE char20
                               CHANGING cv_decimals TYPE i.
*  data lv_amount type char20.
  DATA lv_len TYPE i.
  DATA: lv_decimal TYPE zp33_dec18_5.    "TYPE p DECIMALS 5.
  DATA: lv_int_part TYPE zp33_dec18_5.     "TYPE i.
  DATA: lv_deci_part(7).    ") TYPE p DECIMALS 5.
  DATA: lv_21(7).
  DATA: lv_2(10).

  CLEAR cv_decimals.

  TRY. "C-001 +
      MOVE pv_amount TO lv_decimal.

      lv_int_part = trunc( lv_decimal ).
      lv_deci_part = frac( lv_decimal ).
*
      SPLIT lv_deci_part AT '.' INTO lv_21 lv_2 IN  CHARACTER  MODE.
      lv_len = strlen( lv_2 ).

      cv_decimals = lv_len.
    CATCH cx_root INTO DATA(lx_error)."C-001 +
      MESSAGE e014(zfi_msg) WITH pv_amount gs_upload-row_number '' INTO gv_dummy. "C-001 +
      PERFORM frm_add_error_to_log. "C-001 +
      gv_do_not_process = gc_x. "C-001 +
  ENDTRY."C-001 +
ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_check_accruals
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_check_accruals.
*
  LOOP AT gt_set_values_accrual_docs INTO gs_set_values_accrual_docs
     WHERE  ( from = gs_upload-blart OR to = gs_upload-blart ).
    EXIT.
  ENDLOOP.
*
  IF sy-subrc EQ 0.
*    reversal reason
    IF gs_upload-stgrd IS INITIAL .
      MESSAGE e016(zfi_msg) WITH 'reversal reason' gs_upload-row_number INTO gv_dummy.
* Reversal entries must have & ,on row &
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
*    reversal date
    IF gs_upload-stodt IS INITIAL .
      MESSAGE e016(zfi_msg) WITH 'reversal date' gs_upload-row_number INTO gv_dummy.
* Reversal entries must have & ,on row &
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
  ELSE.    "not accrual document type
*    reversal date
    IF NOT gs_upload-stodt IS INITIAL .
      MESSAGE e054(zfi_msg) WITH gs_upload-blart gs_upload-stodt   INTO gv_dummy.
* Document type &, is not For accruals, Reversal date & , entered
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
  ENDIF.
*
  CHECK gv_do_not_process NE gc_x.
*  "do other checks if there were no errors
*

ENDFORM.
*&---------------------------------------------------------------------*
*& FORM frm_build_document_header.
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_build_document_header.

  DATA lv_date(8).
*
  CLEAR gt_return.
*
  CLEAR gt_upload_store .
  CLEAR gv_posnr.
  CLEAR gs_documentheader.
  CLEAR gt_accountgl.
  CLEAR gt_accountreceivable.
  CLEAR gt_accountpayable.
  CLEAR gt_currencyamount.
  CLEAR gt_extension1.
***BEGIN OF INSERTION CH-1321
  CLEAR: gs_acc_ext, gt_extension2.
***END OF INSERTION CH-1321
*  CLEAR gt_currencyamount_accr.
*  CLEAR gt_accountgl_accr.
**  gs_documentheader-
  gs_documentheader-obj_type =  'BKPFF'.
  gs_documentheader-obj_key  = 'JournalUpload'.
  gs_documentheader-obj_sys = sy-sysid && 'CLNT' && syst-mandt.
**  gs_documentheader-bus_act = 'RFBU'.
  gs_documentheader-username = sy-uname.
  gs_documentheader-header_txt = gs_upload-bktxt.
  gs_documentheader-comp_code = gs_upload-bukrs.
  CLEAR lv_date.
  lv_date = gs_upload-bldat+4(4) && gs_upload-bldat(4).
  gs_documentheader-doc_date = lv_date.
  CLEAR lv_date.
  lv_date = gs_upload-budat+4(4) && gs_upload-budat(4).
  gs_documentheader-pstng_date = lv_date.
*TRANS_DATE
  gs_documentheader-fisc_year = gs_documentheader-pstng_date(4).
  gs_documentheader-fis_period = gs_upload-monat.
  gs_documentheader-doc_type = gs_upload-blart.
  gs_documentheader-ref_doc_no = gs_upload-xblnr.
*AC_DOC_NO
*OBJ_KEY_R
  CLEAR gv_accrual.
  LOOP AT gt_set_values_accrual_docs TRANSPORTING  NO FIELDS WHERE from  =  gs_upload-blart   .
*****************    gv_accrual = gc-x.
    EXIT.
  ENDLOOP.
  IF sy-subrc EQ 0.
*****  IF   gv_accrual = gc-x.
    gs_documentheader-reason_rev = '05'.    "gs_upload-stgrd.
***** Add reversal date
    gs_extension1-field1 = 'STODT'.

    DATA lv_day_in            TYPE sy-datum.
    DATA lv_last_day_of_month TYPE sy-datum.

    lv_day_in = gs_documentheader-doc_date.

    CALL FUNCTION 'RP_LAST_DAY_OF_MONTHS'
      EXPORTING
        day_in            = lv_day_in
      IMPORTING
        last_day_of_month = lv_last_day_of_month
      EXCEPTIONS
        day_in_no_date    = 1
        OTHERS            = 2.
    IF sy-subrc <> 0.
* Implement suitable error handling here
    ENDIF.
    ADD 1 TO lv_last_day_of_month.
    gs_extension1-field1+5(8) =  lv_last_day_of_month.
**    SHIFT gs_extension1-field2 LEFT DELETING LEADING ' '.
    APPEND gs_extension1 TO gt_extension1.
***************    CLEAR : gs_documentheader-header_txt.
  ENDIF.
*COMPO_ACC
*REF_DOC_NO_LONG
* REF_DOC_NO_LONG
  gs_documentheader-acc_principle = gs_upload-ledger_group.
*NEG_POSTNG
  gs_documentheader-obj_key_inv = gs_upload-rebzg.
*BILL_CATEGORY
*VATDATE
*INVOICE_REC_DATE
*ECS_ENV
*PARTIAL_REV
*DOC_STATUS


ENDFORM.


FORM frm_build_currency_data.
  TRY.
      DATA ls_finsc_ld_cmp TYPE finsc_ld_cmp.
      DATA ls_finsc_001a TYPE finsc_001a.

      CLEAR gs_currencyamount.
*"00"
      gs_currencyamount-itemno_acc = gv_posnr.
      gs_currencyamount-curr_type = '00'.   "gs_upload-curtp.
      gs_currencyamount-currency = gs_upload-waers.
      gs_currencyamount-amt_doccur = gs_upload-wrbtr.
      IF gs_tbsl-shkzg = 'H'.
        gs_currencyamount-amt_doccur = gs_currencyamount-amt_doccur  * -1.
      ENDIF.
      ADD gs_currencyamount-amt_doccur TO gv_currencyamount_amt_doccur.
      APPEND gs_currencyamount TO gt_currencyamount.
*******  "10"  column ac
      IF gs_upload-dmbtr  IS NOT INITIAL .
        gs_currencyamount-itemno_acc = gv_posnr.
        gs_currencyamount-curr_type = '10'.   "gs_upload-curtp.
        gs_currencyamount-currency = gs_t001-waers.   "gs_upload-waers.
        gs_currencyamount-amt_doccur = gs_upload-dmbtr.
        IF gs_tbsl-shkzg = 'H'.
          gs_currencyamount-amt_doccur = gs_currencyamount-amt_doccur  * -1.
        ENDIF.
        APPEND gs_currencyamount TO gt_currencyamount.
      ENDIF.
*
**  "30" if column ad is not initial
      CLEAR ls_finsc_001a.

      IF gs_upload-dmbe2  IS NOT INITIAL.   "**get the curtyp from FINSC_LD_CMP
        SELECT SINGLE waers
    FROM finsc_001a
    INTO ls_finsc_001a-waers
    WHERE bukrs = gs_upload-bukrs
        AND curtype = '30'.
        IF ls_finsc_001a-waers NE gs_upload-waers.
          gs_currencyamount-itemno_acc = gv_posnr.
          gs_currencyamount-curr_type = '30'.
          gs_currencyamount-currency = ls_finsc_001a-waers.
          gs_currencyamount-amt_doccur = gs_upload-dmbe2.
          IF gs_tbsl-shkzg = 'H'.   "40==>40
            gs_currencyamount-amt_doccur = gs_currencyamount-amt_doccur  * -1.
          ENDIF.
          APPEND gs_currencyamount TO gt_currencyamount.
        ENDIF.
      ENDIF.
******

      IF gs_upload-dmbe3  IS NOT INITIAL.   "
**get the curtyp from FINSC_LD_CMP
        SELECT SINGLE *
          FROM finsc_ld_cmp
          INTO ls_finsc_ld_cmp
          WHERE rldnr = '0L'    "use constant as per FDS
        AND  bukrs = gs_upload-bukrs.
        DATA ls_t880 TYPE t880.

        IF sy-subrc EQ 0.

          SELECT SINGLE t880~curr                      "#EC CI_BUFFJOIN
      FROM t001 INNER JOIN t880
            ON t001~rcomp = t880~rcomp
      INTO ls_t880-curr
          WHERE t001~bukrs = gs_upload-bukrs .

          IF sy-subrc EQ 0.
**        IF NOT ls_finsc_ld_cmp-curtpo IS INITIAL.
            gs_currencyamount-itemno_acc = gv_posnr.
            gs_currencyamount-curr_type = ls_finsc_ld_cmp-curtpo.
            gs_currencyamount-currency = ls_t880-curr.
            gs_currencyamount-amt_doccur = gs_upload-dmbe3.
            IF gs_tbsl-shkzg = 'H'.
              gs_currencyamount-amt_doccur = gs_currencyamount-amt_doccur  * -1.
            ENDIF.
            APPEND gs_currencyamount TO gt_currencyamount.

          ENDIF.
        ENDIF.
      ENDIF.

    CATCH cx_root INTO DATA(lx_error)."C-001 +
      MESSAGE e000(zfi_msg) WITH lx_error->get_longtext( ). "C-001 +
  ENDTRY."C-001 +
ENDFORM.




FORM frm_build_currency_data_fbb1.

  DATA ls_finsc_ld_cmp TYPE finsc_ld_cmp.
  DATA ls_finsc_001a TYPE finsc_001a.

  CLEAR gs_currencyamount.
*"00"
  gs_currencyamount-itemno_acc = gv_posnr.
  gs_currencyamount-curr_type = '00'.   "gs_upload-curtp.
  gs_currencyamount-currency = gs_upload-waers.
  gs_currencyamount-amt_doccur = gs_upload-wrbtr.
  IF gs_tbsl-shkzg = 'H'.
    gs_currencyamount-amt_doccur = gs_currencyamount-amt_doccur  * -1.
  ENDIF.
  ADD gs_currencyamount-amt_doccur TO gv_currencyamount_amt_doccur.
  APPEND gs_currencyamount TO gt_currencyamount.
**  "10"  column ac
*  IF gs_upload-dmbtr  IS NOT INITIAL . "WVDHEEVER
  gs_currencyamount-itemno_acc = gv_posnr.
  gs_currencyamount-curr_type = '10'.   "gs_upload-curtp.
  gs_currencyamount-currency = gs_t001-waers.   "gs_upload-waers.
  gs_currencyamount-amt_doccur = gs_upload-dmbtr.
  IF gs_tbsl-shkzg = 'H'.
    gs_currencyamount-amt_doccur = gs_currencyamount-amt_doccur  * -1.
  ENDIF.
  APPEND gs_currencyamount TO gt_currencyamount.
*  ENDIF.     "WVDHEEVER
*
**  "30" if column ad is not initial
  CLEAR ls_finsc_001a.

******  IF gs_upload-dmbe2  IS NOT INITIAL.   "**get the curtyp from FINSC_LD_CMP
* WVDHEEVER BEGIN
  SELECT SINGLE waers
    FROM finsc_001a
    INTO ls_finsc_001a-waers
    WHERE bukrs = gs_upload-bukrs
  AND curtype = '30'.
* WVDHEEVER END
***    IF ls_finsc_001a-waers NE gs_upload-waers.
  gs_currencyamount-itemno_acc = gv_posnr.
  gs_currencyamount-curr_type = '30'.
  "  gs_currencyamount-currency = gs_upload-waers.    "WVDHEEVER
  gs_currencyamount-currency = ls_finsc_001a-waers. "WVDHEEVER
  gs_currencyamount-amt_doccur = gs_upload-dmbe2.
  IF gs_tbsl-shkzg = 'H'.   "40==>40
    gs_currencyamount-amt_doccur = gs_currencyamount-amt_doccur  * -1.
  ENDIF.
  APPEND gs_currencyamount TO gt_currencyamount.
***    ENDIF.
******  ENDIF.  IF gs_upload-dmbe2  IS NOT INITIAL.   "**get the curtyp from FINSC_LD_CMP
******
*
**  BREAK mmakda.
  gs_currencyamount-itemno_acc = gv_posnr.
  gs_currencyamount-curr_type = '60'.
  gs_currencyamount-currency = 'CDF'.    "ls_t880-curr.
  gs_currencyamount-amt_doccur = gs_upload-dmbe3.
  IF gs_tbsl-shkzg = 'H'.
    gs_currencyamount-amt_doccur = gs_currencyamount-amt_doccur  * -1.
  ENDIF.
  APPEND gs_currencyamount TO gt_currencyamount.
*
ENDFORM.


FORM frm_get_koart.

  CLEAR gs_tbsl.
  LOOP AT gt_tbsl INTO gs_tbsl WHERE bschl = gs_upload-bschl.
    EXIT.
  ENDLOOP.

ENDFORM.


FORM frm_build_balancing_entry.

  ADD 1 TO  gv_posnr.

  gs_accountgl-itemno_acc = gv_posnr.

  gs_accountgl-gl_account = gv_clearing_gl.

*
  CLEAR gs_accountgl-costcenter.
*
  APPEND gs_accountgl TO gt_accountgl.

  CLEAR gs_currencyamount.
*"00"
  gs_currencyamount-itemno_acc = gv_posnr.
  gs_currencyamount-curr_type = '00'.   "gs_upload-curtp.
  gs_currencyamount-currency = gs_upload-waers.
*
  gv_currencyamount_amt_doccur = gv_currencyamount_amt_doccur * -1.
  gs_currencyamount-amt_doccur = gv_currencyamount_amt_doccur.

  APPEND gs_currencyamount TO gt_currencyamount.

ENDFORM.

*&---------------------------------------------------------------------*
*&      Form  frm_alv_eventtab_build
*&---------------------------------------------------------------------*
*       Prepare ALV Events
*----------------------------------------------------------------------*
FORM frm_alv_eventtab_build USING pt_events TYPE slis_t_event.
*
  DATA: ls_event TYPE slis_alv_event.

  CALL FUNCTION 'REUSE_ALV_EVENTS_GET'
    EXPORTING
      i_list_type = 0
    IMPORTING
      et_events   = pt_events.

  READ TABLE pt_events WITH KEY name =  slis_ev_top_of_page
                                       INTO ls_event.
  IF sy-subrc = 0.
    MOVE 'FRM_ALV_TOP_OF_PAGE' TO ls_event-form.
    MODIFY pt_events FROM ls_event INDEX sy-tabix.
  ENDIF.

ENDFORM.                    "frm_alv_eventtab_build

*&---------------------------------------------------------------------*
*&      Form  frm_alv_top_of_page
*&---------------------------------------------------------------------*
*       FORM TOP_OF_PAGE
*----------------------------------------------------------------------*
FORM frm_alv_top_of_page.                                   "#EC CALLED
*

  CALL FUNCTION 'REUSE_ALV_COMMENTARY_WRITE'
    EXPORTING
      it_list_commentary = gt_alv_list_top_of_page.

ENDFORM.                    "frm_alv_top_of_page

*&---------------------------------------------------------------------*
*&      Form  frm_alv_comment_build_top
*&---------------------------------------------------------------------*
*       Prepare the TOP of ALV
*----------------------------------------------------------------------*
FORM frm_alv_comment_build_top USING pt_top_of_page TYPE slis_t_listheader.

  DATA: ls_line TYPE slis_listheader.
  DATA: lv_datum(10).
  DATA: lv_time(8).

  CLEAR pt_top_of_page.
  CLEAR ls_line.
  ls_line-typ  = 'S'.
  ls_line-key = 'Report Name:'.
  ls_line-info = sy-repid.
  APPEND ls_line TO pt_top_of_page.

  CLEAR ls_line.
  ls_line-typ  = 'S'.
  ls_line-key = 'Report Description:'.
  ls_line-info = sy-title.
  APPEND ls_line TO pt_top_of_page.

  CLEAR ls_line.
  ls_line-typ  = 'S'.
  ls_line-key = 'User ID:'.
  ls_line-info = sy-uname.
  APPEND ls_line TO pt_top_of_page.

  CLEAR ls_line.
  ls_line-typ  = 'S'.
  ls_line-key = 'Date:'.
  WRITE sy-datum TO lv_datum.
  ls_line-info = lv_datum.
  APPEND ls_line TO pt_top_of_page.

  CLEAR ls_line.
  ls_line-typ  = 'S'.
  ls_line-key = 'Time:'.
  WRITE sy-uzeit TO lv_time.
  ls_line-info = lv_time.
  APPEND ls_line TO pt_top_of_page.

  CLEAR ls_line.
  ls_line-typ  = 'S'.
  ls_line-key = 'File:'.
  ls_line-info = p_file.
  APPEND ls_line TO pt_top_of_page.


  CLEAR ls_line.
  ls_line-key = '     '.
  IF p_test = 'X'.
    ls_line-typ  = 'S'.
    ls_line-info = 'Test Run'.
******    IF gv_do_not_process_update = gc-x.
******      ls_line-info = 'Test Run with errors'.
******    ELSE.
******      ls_line-info = 'Test Run without errors'.
******    ENDIF.
**  ls_line-info = p_file.
    APPEND ls_line TO pt_top_of_page.
  ELSE.
    IF gv_do_not_process_update = gc_x.
      ls_line-info = 'Update Run with errors'.
    ELSE.
      ls_line-info = 'Update Run without errors'.
    ENDIF.
    ls_line-typ  = 'S'.
**  ls_line-info = p_file.
    APPEND ls_line TO pt_top_of_page.

  ENDIF.
*  processing option FB01
  ls_line-info = 'Regular journal entry'.

  ls_line-typ  = 'S'.
**  ls_line-info = p_file.
  APPEND ls_line TO pt_top_of_page.
*
ENDFORM.                    "frm_alv_comment_build_top

FORM frm_check_dates.

  PERFORM frm_check_date_plausibilty  USING gs_upload-budat 'Posting Date' .
  PERFORM frm_check_date_plausibilty  USING gs_upload-bldat 'Document Date' .
  PERFORM frm_check_date_plausibilty  USING gs_upload-zfbdt 'Baseline Date'.
  PERFORM frm_check_date_plausibilty  USING gs_upload-stodt 'Reversal date'.

ENDFORM.

FORM frm_check_date_plausibilty USING pv_date TYPE char08 pv_date_type TYPE char30.
  DATA lv_date TYPE soes-jobcount.

  IF NOT pv_date IS INITIAL.
    lv_date = pv_date+4(4) && pv_date(4).

    CALL FUNCTION 'CNV_SDATE_CHECK_PLAUSIBLE'
      EXPORTING
        sdate                     = lv_date
      EXCEPTIONS
        plausibility_check_failed = 1
        OTHERS                    = 2.

    IF sy-subrc <> 0.
      MESSAGE e055(zfi_msg) WITH pv_date pv_date_type   INTO gv_dummy.
* Date & format should be (mmddyyyy) - &
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
  ENDIF.

ENDFORM.



FORM frm_screen_display_by_tcode.
*
*  IF sy-tcode = gc-tcode_fbb1  .
*    LOOP AT SCREEN.
*      IF screen-group1 = 'MG1'.
*        screen-intensified = '1'.
*        screen-display_3d = '1'.
*        screen-color = 5.
****BEGIN OF INSERTION CH-1321
*      ELSEIF screen-group1 = 'FIL'.
*        screen-required = '2'.
****END OF INSERTION CH-1321
*      ENDIF.
*      MODIFY SCREEN.
*    ENDLOOP.
*  ELSE.   "FB01
  LOOP AT SCREEN.
    IF screen-group1 = 'MG1'.
      screen-invisible = '1'.
***BEGIN OF INSERTION CH-1321
    ELSEIF screen-group1 = 'FIL'.
      screen-required = '2'.
***END OF INSERTION CH-1321
    ENDIF.
    MODIFY SCREEN.
  ENDLOOP.
*  ENDIF.
*
ENDFORM.

FORM frm_check_header.

  DATA: lv_ledger_group TYPE accounting_principle,
        lv_bukrs        TYPE bukrs,
        lv_bldat        TYPE bldat,
        lv_budat        TYPE budat,
        lv_monat        TYPE monat,
        lv_blart        TYPE blart,
        lv_waers        TYPE waers,
        lv_xblnr        TYPE xblnr,
        lv_bktxt        TYPE bktxt.

  CLEAR:  lv_ledger_group, lv_bukrs, lv_bldat, lv_budat,
          lv_monat, lv_blart, lv_waers, lv_xblnr, lv_bktxt.
  CLEAR:  gs_header_upload, gs_header_upload2.
  CLEAR:  gt_header_upload[].

  LOOP AT gt_upload INTO gs_header_upload WHERE row_number = gs_upload-row_number.
    MOVE-CORRESPONDING gs_header_upload TO gs_header_upload2.
    APPEND gs_header_upload2 TO gt_header_upload.
    CLEAR: gs_header_upload2.
  ENDLOOP.

  SORT gt_header_upload BY row_number.

  READ TABLE gt_header_upload INTO gs_header_upload2 INDEX 1.

  lv_ledger_group = gs_header_upload2-ledger_group.
  lv_bukrs        = gs_header_upload2-bukrs.
  lv_bldat        = gs_header_upload2-bldat.
  lv_budat        = gs_header_upload2-budat.
  lv_monat        = gs_header_upload2-monat.
  lv_blart        = gs_header_upload2-blart.
  lv_waers        = gs_header_upload2-waers.
  lv_xblnr        = gs_header_upload2-xblnr.
  lv_bktxt        = gs_header_upload2-bktxt.

  LOOP AT gt_header_upload INTO gs_header_upload2 FROM 2.

    IF (  lv_ledger_group = gs_header_upload2-ledger_group
      AND lv_bukrs        = gs_header_upload2-bukrs
      AND lv_bldat        = gs_header_upload2-bldat
      AND lv_budat        = gs_header_upload2-budat
      AND lv_monat        = gs_header_upload2-monat
      AND lv_blart        = gs_header_upload2-blart
      AND lv_waers        = gs_header_upload2-waers
      AND lv_xblnr        = gs_header_upload2-xblnr
      AND lv_bktxt        = gs_header_upload2-bktxt ).

    ELSE.
      MESSAGE e063(zfi_msg) WITH gs_upload-row_number INTO gv_dummy.
      PERFORM frm_add_error_to_log.
      gv_do_not_process = gc_x.
    ENDIF.
  ENDLOOP.

  CLEAR:  lv_ledger_group, lv_bukrs, lv_bldat, lv_budat,
          lv_monat, lv_blart, lv_waers, lv_xblnr, lv_bktxt.
ENDFORM.
*&---------------------------------------------------------------------*
*& Form frm_download_excel
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_download_excel .
  CONSTANTS: lc_extension TYPE string VALUE '.XLSX',
             lc_srtf2_0   TYPE indx_srtf2 VALUE 0,
             lc_relid_mi  TYPE w3_relid VALUE 'MI'.

  DATA: lv_filename    TYPE string,
        lv_path        TYPE string,
        lv_fullpath    TYPE string,
        ls_key         TYPE wwwdatatab,
        lv_destination LIKE rlgrap-filename,
        lv_rc          TYPE sy-subrc.
  DATA lv_default TYPE string.

  DATA lv_objid     TYPE w3objid.

  IF sy-ucomm = 'FC01'. "Download template
    lv_objid = 'ZCFI_JOURNAL'.
    lv_default = |{ TEXT-002 }_{ sy-datum && lc_extension }|.

    "获取下载文件路径
    CALL METHOD cl_gui_frontend_services=>file_save_dialog
      EXPORTING
        window_title              = CONV #( TEXT-001 )
        default_extension         = lc_extension
        default_file_name         = lv_default
        file_filter               = cl_gui_frontend_services=>filetype_excel
      CHANGING
        filename                  = lv_filename
        path                      = lv_path
        fullpath                  = lv_fullpath
      EXCEPTIONS
        cntl_error                = 1
        error_no_gui              = 2
        not_supported_by_gui      = 3
        invalid_default_file_name = 4
        OTHERS                    = 5.

    IF lv_fullpath IS NOT INITIAL.
      "检查模板是否存在
      SELECT SINGLE wwwdata~relid
                    wwwdata~objid
        INTO CORRESPONDING FIELDS OF ls_key
        FROM wwwdata
       WHERE objid = lv_objid
         AND srtf2 = lc_srtf2_0
      AND relid = lc_relid_mi.
      IF sy-subrc <> 0.
        MESSAGE s089(zcsd_msg) WITH lv_objid DISPLAY LIKE 'E'.
      ELSE.
        CLEAR: lv_destination.
        lv_destination = lv_fullpath.
        "下载模板
        CALL FUNCTION 'DOWNLOAD_WEB_OBJECT'
          EXPORTING
            key         = ls_key
            destination = lv_destination
          IMPORTING
            rc          = lv_rc.
        IF lv_rc NE 0.
          MESSAGE s090(zcsd_msg) WITH lv_objid DISPLAY LIKE 'E'.
        ELSE.
          MESSAGE s091(zcsd_msg).
        ENDIF.
      ENDIF.
    ELSE.
      MESSAGE s088(zcsd_msg) DISPLAY LIKE 'E'.
    ENDIF.
  ENDIF.
ENDFORM.
*&---------------------------------------------------------------------*
*& Form frm_check_fbb1_authorisation
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
FORM frm_check_fbb1_authorisation .
  AUTHORITY-CHECK OBJECT 'S_TCODE'
   ID 'TCD' FIELD 'FBB1'.
  IF sy-subrc <> 0.
    MESSAGE e061(zfi_msg).
* You are not authorized to post journals using FBB1
  ENDIF.
ENDFORM.
```
