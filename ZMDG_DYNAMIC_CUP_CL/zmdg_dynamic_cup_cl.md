```abap
CLASS zmdg_dynamic_cup_cl DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.

    INTERFACES if_badi_interface .
    INTERFACES if_usmd_ssw_dynamic_agt_select .
  PROTECTED SECTION.
  PRIVATE SECTION.

    CLASS-METHODS read_material
      IMPORTING
        !iv_cr_number TYPE usmd_crequest
      EXPORTING
        !es_material  TYPE /mdgmm/_s_mm_pp_material .
    CLASS-METHODS read_cr_objectlist
      IMPORTING
        !io_model         TYPE REF TO if_usmd_model_ext
        !iv_crequest_id   TYPE usmd_crequest
      RETURNING
        VALUE(rt_objlist) TYPE usmd_t_crequest_entity .
    CLASS-METHODS read_bp_centrl
      IMPORTING
        !iv_cr_number TYPE usmd_crequest
      EXPORTING
        !es_bp_centrl TYPE /mdgbp/_s_bp_pp_bp_centrl .
    CLASS-METHODS read_bp_sales
      IMPORTING
        !iv_cr_number TYPE usmd_crequest
      EXPORTING
        !et_bp_sales  TYPE zzmdg_bp_sales .
    CLASS-METHODS read_bp_idnum
      IMPORTING
        !iv_cr_number TYPE usmd_crequest
      EXPORTING
        !et_bp_idnum  TYPE zzmdg_bp_idnum
        !ev_partner   TYPE bu_partner .
    CLASS-METHODS read_material_desc
      IMPORTING
        !iv_cr_number TYPE usmd_crequest
      EXPORTING
        !ev_maktx     TYPE maktx .
    CLASS-METHODS check_cr_type_newstep
      IMPORTING
        !iv_cr_type  TYPE usmd_crequest_type
        !iv_new_step TYPE usmd_crequest_appstep
      EXPORTING
        !ev_agt_val  TYPE char10 .
ENDCLASS.



CLASS ZMDG_DYNAMIC_CUP_CL IMPLEMENTATION.


* <SIGNATURE>---------------------------------------------------------------------------------------+
* | Static Private Method ZMDG_DYNAMIC_CUP_CL=>CHECK_CR_TYPE_NEWSTEP
* +-------------------------------------------------------------------------------------------------+
* | [--->] IV_CR_TYPE                     TYPE        USMD_CREQUEST_TYPE
* | [--->] IV_NEW_STEP                    TYPE        USMD_CREQUEST_APPSTEP
* | [<---] EV_AGT_VAL                     TYPE        CHAR10
* +--------------------------------------------------------------------------------------</SIGNATURE>
  METHOD check_cr_type_newstep.
    CLEAR ev_agt_val.
* Check CR new step
    CASE iv_cr_type.
      WHEN 'ZCUP09'.
        CASE iv_new_step.
          WHEN '90'. ev_agt_val = 'AP01'.
          WHEN '93'. ev_agt_val = 'AP02'.
          WHEN '94'. ev_agt_val = 'AP03'.
          WHEN '96'. ev_agt_val = 'AP04'.
          WHEN '98'. ev_agt_val = 'AP04'.
          WHEN OTHERS.
            RETURN.
        ENDCASE.
      WHEN 'ZCUP12' OR 'ZCUP15'.
        CASE iv_new_step.
          WHEN '90'. ev_agt_val = 'AP01'.
          WHEN '93'. ev_agt_val = 'AP03'.
          WHEN '94'. ev_agt_val = 'AP04'.
          WHEN '98'. ev_agt_val = 'AP04'.
          WHEN OTHERS.
            RETURN.
        ENDCASE.
      WHEN 'ZCUP24'.
        CASE iv_new_step.
          WHEN '90'. ev_agt_val = 'AP03'.
          WHEN OTHERS.
            RETURN.
        ENDCASE.
      WHEN OTHERS.
        RETURN.
    ENDCASE.
  ENDMETHOD.


* <SIGNATURE>---------------------------------------------------------------------------------------+
* | Instance Public Method ZMDG_DYNAMIC_CUP_CL->IF_USMD_SSW_DYNAMIC_AGT_SELECT~GET_DYNAMIC_AGENTS
* +-------------------------------------------------------------------------------------------------+
* | [--->] IV_CR_NUMBER                   TYPE        USMD_CREQUEST
* | [--->] IV_SERVICE_NAME                TYPE        USMD_SERVICE_NAME
* | [--->] IV_PAR_AGT_GRP_NUM             TYPE        USMD_AGENT_GROUP(optional)
* | [<---] ET_MESSAGE                     TYPE        USMD_T_MESSAGE
* | [<-->] CT_NON_USER_AGENT_GROUP        TYPE        USMD_T_NON_USER_AGENT_GROUP
* | [<-->] CV_EXP_COMP_HOURS              TYPE        INT2
* | [<-->] CT_USER_AGENT_GROUP            TYPE        USMD_T_USER_AGENT_GROUP
* | [<-->] CT_CONTEXT_TAB                 TYPE        USMD_T_GENERIC_CONTEXT
* | [<-->] CV_NEW_STEP                    TYPE        USMD_CREQUEST_APPSTEP
* | [<-->] CV_NEW_CR_STATUS               TYPE        USMD_CREQUEST_STATUS
* | [<-->] CV_MERGE_TYPE                  TYPE        USMD_STATUS_MERGE_TYPE
* | [<-->] CV_MERGE_PARAM                 TYPE        USMD_SERVICE_NAME
* +--------------------------------------------------------------------------------------</SIGNATURE>
  METHOD if_usmd_ssw_dynamic_agt_select~get_dynamic_agents.
* Get the users from a differemt source than the "user agent" decision table
    DATA: lt_bp_sales         TYPE zzmdg_bp_sales.
    DATA: lv_agt_val          TYPE char10.
    DATA: ls_user_agent_group TYPE usmd_s_user_agent_group.
    DATA: ls_user_agent       TYPE usmd_s_user_agent.
    DATA: ls_material         TYPE /mdgmm/_s_mm_pp_material. "#EC NEEDED

* 使用SAP_WFRT user

* Clear exporting parameter
    CLEAR et_message.

* Read cr status
    SELECT SINGLE *
        FROM usmd120c
        INTO @DATA(ls_usmd120c)
        WHERE usmd_crequest = @iv_cr_number.

* Read data model
    SELECT SINGLE usmd_model
        FROM usmd110c
        INTO @DATA(lv_model)
        WHERE usmd_creq_type = @ls_usmd120c-usmd_creq_type.

* Check CR type and CR new step
    check_cr_type_newstep(
          EXPORTING
            iv_cr_type = ls_usmd120c-usmd_creq_type
            iv_new_step = cv_new_step
          IMPORTING
            ev_agt_val = lv_agt_val
    ).
    CHECK lv_agt_val IS NOT INITIAL.

    CASE iv_service_name.
      WHEN 'ZMDG_CUP_DYNAGENT'.
        CLEAR: ct_user_agent_group.

        read_bp_sales(
          EXPORTING
            iv_cr_number = iv_cr_number                 " 变更请求
          IMPORTING
            et_bp_sales = lt_bp_sales                 " Source Structure for PP Mapping
        ).

        "根据公司主体设置审批节点
        READ TABLE lt_bp_sales WITH KEY vkorg = '1001' TRANSPORTING NO FIELDS.
        IF sy-subrc EQ 0.
          "1. 苏州众合
          DATA(lv_vkorg) = '1001'.
        ELSE.
          "2. 股份 - 默认
          lv_vkorg = '1000'.
        ENDIF.

        ls_user_agent_group-agent_group = '001'.
        ls_user_agent_group-step_type = '2'.
        ls_user_agent_group-user_agent = VALUE #( ( user_type = 'AG'
                                                    " JS_L_MDG_CU_1000_AP01
                                                    user_value = |JS_L_MDG_CU_{ lv_vkorg }_{ lv_agt_val }| ) ).
        INSERT ls_user_agent_group INTO TABLE ct_user_agent_group.
      WHEN OTHERS.
        RETURN.
    ENDCASE.
  ENDMETHOD.


* <SIGNATURE>---------------------------------------------------------------------------------------+
* | Static Private Method ZMDG_DYNAMIC_CUP_CL=>READ_BP_CENTRL
* +-------------------------------------------------------------------------------------------------+
* | [--->] IV_CR_NUMBER                   TYPE        USMD_CREQUEST
* | [<---] ES_BP_CENTRL                   TYPE        /MDGBP/_S_BP_PP_BP_CENTRL
* +--------------------------------------------------------------------------------------</SIGNATURE>
  METHOD read_bp_centrl.
    DATA: lr_model    TYPE REF TO if_usmd_model_ext.
    DATA: lt_sel      TYPE usmd_ts_sel.
    DATA: ls_sel      TYPE usmd_s_sel.
    DATA: lt_objlist  TYPE usmd_t_crequest_entity.
    DATA: ls_objlist  TYPE usmd_s_crequest_entity.
    DATA: lv_bp    TYPE bu_partner.
    DATA: lt_bp_centrl TYPE TABLE OF /mdgbp/_s_bp_pp_bp_centrl.
    CONSTANTS: lc_incl  TYPE ddsign   VALUE 'I'.
    CONSTANTS: lc_equal TYPE ddoption VALUE 'EQ'.

    CLEAR es_bp_centrl.
* Get read-only access to USMD model data
    CALL METHOD cl_usmd_model_ext=>get_instance
      EXPORTING
        i_usmd_model = 'BP'
      IMPORTING
        eo_instance  = lr_model.

    ls_sel-sign      = lc_incl.
    ls_sel-option    = lc_equal.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = usmd0_cs_fld-crequest
        it_sel            = lt_sel
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = lt_objlist.
    READ TABLE lt_objlist INTO ls_objlist WITH KEY usmd_entity = 'BP_HEADER'.
    ASSERT sy-subrc = 0.
    lv_bp = ls_objlist-usmd_value.

    CLEAR lt_sel.
    ls_sel-fieldname = 'BP_HEADER'.
    ls_sel-sign   = lc_incl.
    ls_sel-option = lc_equal.
    ls_sel-low    = lv_bp.
    INSERT ls_sel INTO TABLE lt_sel.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = 'BP_CENTRL'
        it_sel            = lt_sel
        i_readmode        = if_usmd_model_ext=>gc_readmode_default
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = lt_bp_centrl.
    READ TABLE lt_bp_centrl INTO es_bp_centrl INDEX 1.
  ENDMETHOD.


* <SIGNATURE>---------------------------------------------------------------------------------------+
* | Static Private Method ZMDG_DYNAMIC_CUP_CL=>READ_BP_IDNUM
* +-------------------------------------------------------------------------------------------------+
* | [--->] IV_CR_NUMBER                   TYPE        USMD_CREQUEST
* | [<---] ET_BP_IDNUM                    TYPE        ZZMDG_BP_IDNUM
* | [<---] EV_PARTNER                     TYPE        BU_PARTNER
* +--------------------------------------------------------------------------------------</SIGNATURE>
  METHOD read_bp_idnum.
    DATA: lr_model    TYPE REF TO if_usmd_model_ext.
    DATA: lt_sel      TYPE usmd_ts_sel.
    DATA: ls_sel      TYPE usmd_s_sel.
    DATA: lt_objlist  TYPE usmd_t_crequest_entity.
    DATA: ls_objlist  TYPE usmd_s_crequest_entity.
    DATA: lv_bp    TYPE bu_partner.
    CONSTANTS: lc_incl  TYPE ddsign   VALUE 'I'.
    CONSTANTS: lc_equal TYPE ddoption VALUE 'EQ'.

    CLEAR et_bp_idnum.

* Get read-only access to USMD model data
    CALL METHOD cl_usmd_model_ext=>get_instance
      EXPORTING
        i_usmd_model = 'BP'
      IMPORTING
        eo_instance  = lr_model.

    ls_sel-sign      = lc_incl.
    ls_sel-option    = lc_equal.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = usmd0_cs_fld-crequest
        it_sel            = lt_sel
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = lt_objlist.
    READ TABLE lt_objlist INTO ls_objlist WITH KEY usmd_entity = 'BP_HEADER'.
    ASSERT sy-subrc = 0.
    lv_bp = ls_objlist-usmd_value.
    ev_partner = lv_bp.

    CLEAR lt_sel.
    ls_sel-fieldname = 'BP_HEADER'.
    ls_sel-sign   = lc_incl.
    ls_sel-option = lc_equal.
    ls_sel-low    = lv_bp.
    INSERT ls_sel INTO TABLE lt_sel.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = 'BP_IDNUM'
        it_sel            = lt_sel
        i_readmode        = if_usmd_model_ext=>gc_readmode_default
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = et_bp_idnum.
  ENDMETHOD.


* <SIGNATURE>---------------------------------------------------------------------------------------+
* | Static Private Method ZMDG_DYNAMIC_CUP_CL=>READ_BP_SALES
* +-------------------------------------------------------------------------------------------------+
* | [--->] IV_CR_NUMBER                   TYPE        USMD_CREQUEST
* | [<---] ET_BP_SALES                    TYPE        ZZMDG_BP_SALES
* +--------------------------------------------------------------------------------------</SIGNATURE>
  METHOD read_bp_sales.
    DATA: lr_model    TYPE REF TO if_usmd_model_ext.
    DATA: lt_sel      TYPE usmd_ts_sel.
    DATA: ls_sel      TYPE usmd_s_sel.
    DATA: lt_objlist  TYPE usmd_t_crequest_entity.
    DATA: ls_objlist  TYPE usmd_s_crequest_entity.
    DATA: lv_bp    TYPE bu_partner.
    CONSTANTS: lc_incl  TYPE ddsign   VALUE 'I'.
    CONSTANTS: lc_equal TYPE ddoption VALUE 'EQ'.

    CLEAR et_bp_sales.

* Get read-only access to USMD model data
    CALL METHOD cl_usmd_model_ext=>get_instance
      EXPORTING
        i_usmd_model = 'BP'
      IMPORTING
        eo_instance  = lr_model.

    ls_sel-sign      = lc_incl.
    ls_sel-option    = lc_equal.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = usmd0_cs_fld-crequest
        it_sel            = lt_sel
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = lt_objlist.
    READ TABLE lt_objlist INTO ls_objlist WITH KEY usmd_entity = 'BP_HEADER'.
    ASSERT sy-subrc = 0.
    lv_bp = ls_objlist-usmd_value.

    CLEAR lt_sel.
    ls_sel-fieldname = 'BP_HEADER'.
    ls_sel-sign   = lc_incl.
    ls_sel-option = lc_equal.
    ls_sel-low    = lv_bp.
    INSERT ls_sel INTO TABLE lt_sel.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = 'BP_SALES'
        it_sel            = lt_sel
        i_readmode        = if_usmd_model_ext=>gc_readmode_default
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = et_bp_sales.
  ENDMETHOD.


* <SIGNATURE>---------------------------------------------------------------------------------------+
* | Static Private Method ZMDG_DYNAMIC_CUP_CL=>READ_CR_OBJECTLIST
* +-------------------------------------------------------------------------------------------------+
* | [--->] IO_MODEL                       TYPE REF TO IF_USMD_MODEL_EXT
* | [--->] IV_CREQUEST_ID                 TYPE        USMD_CREQUEST
* | [<-()] RT_OBJLIST                     TYPE        USMD_T_CREQUEST_ENTITY
* +--------------------------------------------------------------------------------------</SIGNATURE>
  METHOD read_cr_objectlist.
    DATA lt_sel  TYPE usmd_ts_sel.
    DATA ls_sel  TYPE usmd_s_sel.
    CONSTANTS: lc_incl  TYPE ddsign   VALUE 'I'.
    CONSTANTS: lc_equal TYPE ddoption VALUE 'EQ'.

    ls_sel-sign      = lc_incl.
    ls_sel-option    = lc_equal.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_crequest_id.
    INSERT ls_sel INTO TABLE lt_sel.

    CALL METHOD io_model->read_char_value
      EXPORTING
        i_fieldname       = usmd0_cs_fld-crequest
        it_sel            = lt_sel
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = rt_objlist.
  ENDMETHOD.


* <SIGNATURE>---------------------------------------------------------------------------------------+
* | Static Private Method ZMDG_DYNAMIC_CUP_CL=>READ_MATERIAL
* +-------------------------------------------------------------------------------------------------+
* | [--->] IV_CR_NUMBER                   TYPE        USMD_CREQUEST
* | [<---] ES_MATERIAL                    TYPE        /MDGMM/_S_MM_PP_MATERIAL
* +--------------------------------------------------------------------------------------</SIGNATURE>
  METHOD read_material.
    DATA: lr_model    TYPE REF TO if_usmd_model_ext.
    DATA: lt_sel      TYPE usmd_ts_sel.
    DATA: ls_sel      TYPE usmd_s_sel.
    DATA: lt_objlist  TYPE usmd_t_crequest_entity.
    DATA: ls_objlist  TYPE usmd_s_crequest_entity.
    DATA: lv_matnr    TYPE matnr.
    DATA: lr_data     TYPE REF TO data.
    FIELD-SYMBOLS: <lt_data> TYPE SORTED TABLE.
    FIELD-SYMBOLS: <ls_data> TYPE any.
    CONSTANTS: lc_incl  TYPE ddsign   VALUE 'I'.
    CONSTANTS: lc_equal TYPE ddoption VALUE 'EQ'.

    CLEAR: es_material.
* Get read-only access to USMD model data
    CALL METHOD cl_usmd_model_ext=>get_instance
      EXPORTING
        i_usmd_model = if_mdg_bs_mat_gen_c=>gc_model_mm
      IMPORTING
        eo_instance  = lr_model.
* Read object list of CR and get the one and only material:
* Get the key of the (type 1) entity MATERIAL
* from the object list of this CR
    ls_sel-sign      = lc_incl.
    ls_sel-option    = lc_equal.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = usmd0_cs_fld-crequest
        it_sel            = lt_sel
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = lt_objlist.
    READ TABLE lt_objlist INTO ls_objlist INDEX 1.
    ASSERT sy-subrc = 0. " CR not found or contains no material
    lv_matnr = ls_objlist-usmd_value.
* Prepare result table for MATERIAL read
    CALL METHOD lr_model->create_data_reference
      EXPORTING
        i_fieldname = if_mdg_bs_mat_gen_c=>gc_fieldname_material
        i_struct    = lr_model->gc_struct_key_attr
        if_table    = abap_true
        i_tabtype   = lr_model->gc_tabtype_sorted
      IMPORTING
        er_data     = lr_data.
    ASSIGN lr_data->* TO <lt_data>.
* Read MATERIAL via material number and Change Request ID
    CLEAR lt_sel.
    ls_sel-fieldname = if_mdg_bs_mat_gen_c=>gc_fieldname_material.
    ls_sel-sign   = lc_incl.
    ls_sel-option = lc_equal.
    ls_sel-low    = lv_matnr.
    INSERT ls_sel INTO TABLE lt_sel.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = if_mdg_bs_mat_gen_c=>gc_fieldname_material
        it_sel            = lt_sel
        i_readmode        = if_usmd_model_ext=>gc_readmode_default
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = <lt_data>.
* Return the one and only result
    READ TABLE <lt_data> ASSIGNING <ls_data> INDEX 1.
    ASSERT sy-subrc = 0. " CR not found or contains no material
    MOVE-CORRESPONDING <ls_data> TO es_material.
  ENDMETHOD.


* <SIGNATURE>---------------------------------------------------------------------------------------+
* | Static Private Method ZMDG_DYNAMIC_CUP_CL=>READ_MATERIAL_DESC
* +-------------------------------------------------------------------------------------------------+
* | [--->] IV_CR_NUMBER                   TYPE        USMD_CREQUEST
* | [<---] EV_MAKTX                       TYPE        MAKTX
* +--------------------------------------------------------------------------------------</SIGNATURE>
  METHOD read_material_desc.
    DATA: lr_model    TYPE REF TO if_usmd_model_ext.
    DATA: lt_sel      TYPE usmd_ts_sel.
    DATA: ls_sel      TYPE usmd_s_sel.
    DATA: lt_objlist  TYPE usmd_t_crequest_entity.
    DATA: ls_objlist  TYPE usmd_s_crequest_entity.
    DATA: lv_matnr    TYPE matnr.
    DATA: lt_material_text TYPE TABLE OF /mdgmm/_st_mm_pp_material.
    CONSTANTS: lc_incl  TYPE ddsign   VALUE 'I'.
    CONSTANTS: lc_equal TYPE ddoption VALUE 'EQ'.

    CLEAR ev_maktx.

* Get read-only access to USMD model data
    CALL METHOD cl_usmd_model_ext=>get_instance
      EXPORTING
        i_usmd_model = 'MM'
      IMPORTING
        eo_instance  = lr_model.

    ls_sel-sign      = lc_incl.
    ls_sel-option    = lc_equal.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = usmd0_cs_fld-crequest
        it_sel            = lt_sel
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = lt_objlist.
    READ TABLE lt_objlist INTO ls_objlist INDEX 1.
    ASSERT sy-subrc = 0.
    lv_matnr = ls_objlist-usmd_value.

    CLEAR lt_sel.
    ls_sel-fieldname = 'MATERIAL'.
    ls_sel-sign   = lc_incl.
    ls_sel-option = lc_equal.
    ls_sel-low    = lv_matnr.
    INSERT ls_sel INTO TABLE lt_sel.
    ls_sel-fieldname = usmd0_cs_fld-crequest.
    ls_sel-low       = iv_cr_number.
    INSERT ls_sel INTO TABLE lt_sel.
    CALL METHOD lr_model->read_char_value
      EXPORTING
        i_fieldname       = 'MATERIAL'
        it_sel            = lt_sel
        i_readmode        = if_usmd_model_ext=>gc_readmode_default
        if_use_edtn_slice = abap_false
      IMPORTING
        et_data           = lt_material_text.

    READ TABLE lt_material_text INTO DATA(ls_material_text) WITH KEY langu = '1'.
    IF sy-subrc EQ 0.
      ev_maktx = ls_material_text-txtmi.
    ENDIF.
  ENDMETHOD.
ENDCLASS.
```