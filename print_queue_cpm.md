Print Queue + CPM 
注意点：Print Queue里面 default打印队列不会触发打印，需要新建打印队列。

定义一个class处理数据插入 print Queue，这步必须是异步的，要不然会报错。

CLASS zcl_bg_print_worker DEFINITION

  PUBLIC
  FINAL
  CREATE PUBLIC.

  PUBLIC SECTION.

    INTERFACES if_bgmc_op_single_tx_uncontr.

    TYPES:
      BEGIN OF ty_print_param,
        serial_no TYPE string,
      END OF ty_print_param.

    METHODS constructor
      IMPORTING
        is_param TYPE ty_print_param.

  PRIVATE SECTION.

    DATA ms_param TYPE ty_print_param.

  ENDCLASS.



CLASS zcl_bg_print_worker IMPLEMENTATION.


  METHOD constructor.

    ms_param = is_param.

  ENDMETHOD.



  METHOD if_bgmc_op_single_tx_uncontr~execute.

    TRY.

        "============================================================
        " 1. 获取 FDP Service
        "============================================================

        DATA(lo_fdp_api) =
          cl_fp_fdp_services=>get_instance(
            `ZPP_SD_FORM_LABEL`
          ).


        "============================================================
        " 2. 获取 FDP Keys
        "============================================================

        DATA(lt_keys) =
          lo_fdp_api->get_keys( ).


        "============================================================
        " 3. 设置 SerialNo
        "============================================================

        lt_keys[
          name = 'SERIALNO'
        ]-value =
          CONV string( ms_param-serial_no ).


        "============================================================
        " 4. 获取 Form XML
        "============================================================

        DATA(lv_xml_data) =
          lo_fdp_api->read_to_xml_v2(
            lt_keys
          ).


        "============================================================
        " 5. 获取 Adobe Form XDP
        "============================================================

        DATA(lo_form_reader) =
          cl_fp_form_reader=>create_form_reader(
            'ZFORM_BG_LABEL'
          ).


        DATA(lv_xdp_binary) =
          lo_form_reader->get_layout( ).


        "============================================================
        " 6. ADS Render
        "
        " 注意：
        " 这里现在是在 BGMC Transactional Uncontrolled
        " background process 中执行。
        "
        " 因此不再触发 RAP BEHAVIOR_ILLEGAL_STATEMENT。
        "============================================================

        DATA lv_locale TYPE string VALUE 'zh_CN'.

        DATA lv_pdl_binary TYPE xstring.

        DATA lv_pages TYPE int4.


        cl_fp_ads_util=>render_4_pq(
          EXPORTING
            iv_xml_data = lv_xml_data
            iv_pq_name = 'PRINT_CONFIRMATION'
            iv_xdp_layout = lv_xdp_binary
            iv_locale = lv_locale
          IMPORTING
            ev_pdl = lv_pdl_binary
            ev_pages = lv_pages

        ).


        "============================================================
        " 7. 创建 Print Queue Item
        "============================================================

        DATA:
          lv_assigned_job_id
            TYPE cl_print_queue_utils=>ty_itemid,
          lv_spool_error_txt
            TYPE cl_print_queue_utils=>ty_msg.

        lv_assigned_job_id =
          cl_print_queue_utils=>create_queue_item_by_data(
            EXPORTING
              iv_qname = 'PRINT_CONFIRMATION'
              iv_print_data = lv_pdl_binary
              iv_name_of_main_doc = |Confirmation_{ ms_param-serial_no }|
              iv_number_of_copies = 1
              iv_pages = lv_pages
            IMPORTING
              ev_err_msg = lv_spool_error_txt

          ).


        "============================================================
        " 8. 更新打印结果
        "============================================================

        IF lv_assigned_job_id IS NOT INITIAL
           AND lv_spool_error_txt IS INITIAL.

          UPDATE zdt_bg_prtlabel
             SET zlast_print_status = 'SUCCESS',
                 znote_no = ''
           WHERE zserialno =
                 @ms_param-serial_no.


        ELSE.

          UPDATE zdt_bg_prtlabel
             SET zlast_print_status = 'ERROR',
                 znote_no = @lv_spool_error_txt
           WHERE zserialno = @ms_param-serial_no.


        ENDIF.


      CATCH cx_fp_fdp_error
            cx_fp_form_reader
            cx_fp_ads_util
            INTO DATA(lx_print).

        DATA(lv_print_error) =
          lx_print->get_text( ).

        UPDATE zdt_bg_prtlabel
           SET zlast_print_status =
                 'ERROR',
               znote_no =
                 @lv_print_error
         WHERE zserialno =
               @ms_param-serial_no.

        " BGMC operation 失败
        " 让 BGMC framework 记录 operation failure

        RAISE EXCEPTION TYPE cx_bgmc_operation
          EXPORTING
            previous       = lx_print
            textid         = cx_bgmc_operation=>t100_operation_failed
            retry_settings = VALUE #(
                               do_retry    = abap_false
                               delay_time  = 0
                             ).
      CATCH cx_root INTO DATA(lx_root).
        DATA(lv_error_text) =
          lx_root->get_text( ).
        UPDATE zdt_bg_prtlabel
           SET zlast_print_status =
                 'ERROR',
               znote_no =
                 @lv_error_text
         WHERE zserialno =
               @ms_param-serial_no.
        RAISE EXCEPTION TYPE cx_bgmc_operation
          EXPORTING
            previous       = lx_root
            textid         = cx_bgmc_operation=>t100_operation_failed
            retry_settings = VALUE #(
                               do_retry    = abap_false
                               delay_time  = 0
                             ).
    ENDTRY.

  ENDMETHOD.

ENDCLASS.

新增一个action printLabel 来处理打印。全部代码在右边

METHOD printlabel.
************************* Important  *********************************
***  这个method打印标签是通过发送CPM 实现的功能，现在该功能废弃改为前端打印。
**********************************************************************

    READ TABLE keys INTO DATA(ls_key) INDEX 1.

    IF sy-subrc <> 0
       OR ls_key-%tky IS INITIAL.

      RETURN.

    ENDIF.


    DATA(lv_target_serial) = CONV string( ls_key-serialno ).

    IF lv_target_serial IS INITIAL.

      RETURN.

    ENDIF.


    TRY.

        "------------------------------------------------------------
        " 1. 创建 BGMC Operation
        "------------------------------------------------------------

        DATA lo_operation TYPE REF TO if_bgmc_op_single_tx_uncontr.

        lo_operation = NEW zcl_bg_print_worker(
          VALUE #(
            serial_no = lv_target_serial
          )
        ).


        "------------------------------------------------------------
        " 2. 创建 BGMC Process Factory
        "------------------------------------------------------------

        DATA(lo_factory) =
          cl_bgmc_process_factory=>get_default( ).


*        "------------------------------------------------------------
*        " 3. 创建 Transactional Background Process
*        "------------------------------------------------------------
*
*        DATA(lo_process) =
*          lo_factory->create( ).


        "------------------------------------------------------------
        " 4. 设置 Process 名称 + Operation
        "
        " 注意：
        " 不能使用 request_operation
        " 不能使用 set_operation
        " 必须使用 set_operation_tx_uncontrolled
        " 因为打印 API 内部需要 COMMIT WORK
        "------------------------------------------------------------

        DATA(lo_process) =
          lo_factory->create( ).

        lo_process->set_name(
          |ZBAOGONG_Print_{ lv_target_serial }|
        ).

        lo_process->set_operation_tx_uncontrolled(
          lo_operation
        ).


        "------------------------------------------------------------
        " 5. 保存 BGMC Process 到本次 RAP LUW
        "
        " 这里先不执行。
        " save_for_execution() 放到 additional save。
        "------------------------------------------------------------

        lcl_buffer=>go_bgmc_process = lo_process.


        "------------------------------------------------------------
        " 6. 更新打印状态
        "------------------------------------------------------------

        SELECT SINGLE zprint_count
          FROM zdt_bg_prtlabel
          WHERE zserialno = @ls_key-serialno
          INTO @DATA(lv_current_count).


        DATA(lv_next_count) =
          COND i(
            WHEN sy-subrc = 0
            THEN lv_current_count + 1
            ELSE 1
          ).
        UPDATE zdt_bg_prtlabel
             SET zprint_count = @lv_next_count
           WHERE zserialno = @ls_key-serialno .

      CATCH cx_bgmc INTO DATA(lx_bgmc).

        CLEAR lcl_buffer=>go_bgmc_process.

        APPEND VALUE #(
          %tky = ls_key-%tky

          %msg =
            new_message_with_text(
              severity =
                if_abap_behv_message=>severity-error

              text =
                |BGMC创建失败: { lx_bgmc->get_text( ) }|
            )
        )
        TO reported-zi_dt_bg_prtlabel.


        APPEND VALUE #(
          %tky = ls_key-%tky
        )
        TO failed-zi_dt_bg_prtlabel.


      CATCH cx_root INTO DATA(lx_root).

        CLEAR lcl_buffer=>go_bgmc_process.

        APPEND VALUE #(
          %tky = ls_key-%tky

          %msg =
            new_message_with_text(
              severity =
                if_abap_behv_message=>severity-error

              text =
                |打印后台任务创建失败: { lx_root->get_text( ) }|
            )
        )
        TO reported-zi_dt_bg_prtlabel.


        APPEND VALUE #(
          %tky = ls_key-%tky
        )
        TO failed-zi_dt_bg_prtlabel.

    ENDTRY.

  ENDMETHOD.

