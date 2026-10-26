资料链接：CPM: https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/cloud-print-manager-for-pull-integration-installation-and-configuration/ba-p/13334437

这里的url,user,password 都必须是 API的，不是business user



SAP Sample : https://github.com/SAP-samples/abap-partner-reference-application/blob/main/Tutorials/41a-Forms-Feature.md

视频Demo资料： https://community.sap.com/t5/technology-blog-posts-by-sap/sap-forms-service-by-adobe-in-sap-btp-abap-environment/bc-p/13958740

多个PDF合并成一个PDF文档： https://help.sap.com/docs/SAP_S4HANA_CLOUD/6aa39f1ac05441e5a23f484f31e477e7/f647ef4602dc461488cc82444141b51a.html?locale=en-US
<img width="599" height="774" alt="image" src="https://github.com/user-attachments/assets/a8d3a173-ede2-4dee-a864-6de411dadec2" />


printLabelInFront：临时存储pdf
printLabelInFront_backup:直接发送pdf文件流
printLabel：使用Print Queue + CPM


" 1. Update the Memory Buffer Class to cache the active BGMC process handles array
CLASS lcl_buffer DEFINITION.

  PUBLIC SECTION.

    CLASS-DATA:
      go_bgmc_process TYPE REF TO if_bgmc_process_single_op.

ENDCLASS.

CLASS lhc_ZI_DT_BG_PRTLABEL DEFINITION INHERITING FROM cl_abap_behavior_handler.


  PRIVATE SECTION.
    TYPES: BEGIN OF send_response,
             status       TYPE zdt_iflog-status,
             message      TYPE zdt_iflog-message,
             responsejson TYPE string,
           END OF send_response.
    METHODS get_instance_authorizations FOR INSTANCE AUTHORIZATION
      IMPORTING keys REQUEST requested_authorizations FOR zi_dt_bg_prtlabel RESULT result.

*    METHODS get_global_authorizations FOR GLOBAL AUTHORIZATION
*      IMPORTING REQUEST requested_authorizations FOR zi_dt_bg_prtlabel RESULT result.

    METHODS earlynumbering_create FOR NUMBERING
      IMPORTING entities FOR CREATE zi_dt_bg_prtlabel.

    METHODS printlabel FOR MODIFY
      IMPORTING keys FOR ACTION zi_dt_bg_prtlabel~printlabel .

    METHODS postConfirmation FOR MODIFY
      IMPORTING keys FOR ACTION zi_dt_bg_prtlabel~postConfirmation .

    METHODS printLabelInFront FOR MODIFY
      IMPORTING keys   FOR ACTION zi_dt_bg_prtlabel~printLabelInFront
      RESULT    result.

    METHODS calculateFields FOR DETERMINE ON MODIFY
      IMPORTING keys FOR zi_dt_bg_prtlabel~calculateFields.
ENDCLASS.

CLASS lhc_ZI_DT_BG_PRTLABEL IMPLEMENTATION.

  METHOD get_instance_authorizations.
  ENDMETHOD.
*
*  METHOD get_global_authorizations.
*    result-%create = if_abap_behv=>auth-allowed.
*  ENDMETHOD.

  METHOD earlynumbering_create.
    DATA: lv_max_serial   TYPE c LENGTH 10,
          lv_numeric_part TYPE string,
          lv_num          TYPE i,
          lv_next_number  TYPE i,
          lv_new_serial   TYPE c LENGTH 10,
          lv_char_num     TYPE c LENGTH 10.
    TRY.
        " 1. 获取公有云标准锁对象实例 (传入您创建的锁对象名称)
        DATA(lo_lock) = cl_abap_lock_object_factory=>get_instance( iv_name = 'EZDT_BG_PRTLABEL' ).

        " 2. 执行加锁。主键值 ZSERIALNO 留空，代表锁定该表的当前 client，阻止并发冲突。
        lo_lock->enqueue(
          it_parameter = VALUE #( ( name = 'ZSERIALNO' value = REF #( space ) ) )
*         iv_mode      = 'E' " 默认为排他锁
        ).

      CATCH cx_abap_foreign_lock cx_abap_lock_failure INTO DATA(lx_lock_err).
        " 如果加锁失败（被其他并发用户占用），向 RAP 框架汇报失败
        LOOP AT entities INTO DATA(ls_ent).
          APPEND VALUE #( %cid = ls_ent-%cid ) TO failed-zi_dt_bg_prtlabel.

          " Optional: Add a friendly error message to the user interface via reported
          APPEND VALUE #( %cid     = ls_ent-%cid
                          %element-serialno = if_abap_behv=>mk-on
                          %msg     = new_message_with_text(
                                       severity = if_abap_behv_message=>severity-error
                                       text     = 'The number range is currently locked by another user. Please try again.'
                                     ) ) TO reported-zi_dt_bg_prtlabel.
        ENDLOOP.
        RETURN.
    ENDTRY.

    " 1. 获取当前数据库中最大的流水号
    SELECT MAX( zserialno ) FROM zdt_bg_prtlabel INTO @lv_max_serial.

    IF lv_max_serial IS NOT INITIAL AND lv_max_serial+0(1) = 'S'.
      " 2. 提取 S 后面的所有字符，并去掉可能存在的右侧空格
      lv_numeric_part = lv_max_serial+1.
      CONDENSE lv_numeric_part NO-GAPS.

      " 3. 稳健地将紧凑文本转换为纯数字进行 +1 递增
      IF lv_numeric_part CO '0123456789' AND lv_numeric_part IS NOT INITIAL.
        lv_num = lv_numeric_part.
        lv_next_number = lv_num + 1.
      ELSE.
        lv_next_number = 1.
      ENDIF.
    ELSE.
      " 初始化
      lv_next_number = 700000.
    ENDIF.

    " 4. 循环处理前端传来的新建请求
    LOOP AT entities INTO DATA(ls_entity).

      " 5.转换为左对齐的不补零纯文本字符串（例如 "1", "700000"）
      lv_char_num = lv_next_number.
      CONDENSE lv_char_num NO-GAPS.

      " 6. 直接拼接 S，生成如 S1, S2 ... S700000
      lv_new_serial = |S{ lv_char_num }|.

      APPEND VALUE #( %cid      = ls_entity-%cid
                      SerialNo  = lv_new_serial ) TO mapped-zi_dt_bg_prtlabel.

      lv_next_number = lv_next_number + 1.

    ENDLOOP.


    TRY.
        lo_lock->dequeue(
          it_parameter = VALUE #( ( name = 'ZSERIALNO' value = REF #( space ) ) )
        ).
      CATCH cx_abap_lock_failure.
        " If unlocking fails, we ignore it because the system will clear it anyway when the LUW ends
        CLEAR lv_max_serial.
    ENDTRY.
  ENDMETHOD.

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

  METHOD postconfirmation.
    " 读取当前行隐式透传过来的主键记录
    READ TABLE keys INTO DATA(ls_key) INDEX 1.
    IF sy-subrc = 0 AND ls_key-serialno IS NOT INITIAL.
      DATA(lv_serial) = ls_key-serialno.

      " 从自建表中完整拉取最新扩充业务参数
      READ ENTITIES OF zi_dt_bg_prtlabel IN LOCAL MODE
        ENTITY zi_dt_bg_prtlabel
        FIELDS ( ProductionOrder ProductionOrderOperation Product QuantityPerContainer Meins NoteNo
                 Containers StorageLocation OperationControlProfile ProductionOrderItem ProductionPlant GroupUUID )
        WITH VALUE #( ( SerialNo = lv_serial ) )
        RESULT DATA(lt_saved_records).

      READ TABLE lt_saved_records INTO DATA(ls_data) INDEX 1.
      IF sy-subrc = 0.
        TRY.
            " 实例化自定义 HTTP 通信工具类
            DATA: lo_es TYPE REF TO zcl_http_send_es.
            lo_es = NEW zcl_http_send_es( ).

            " 定义通信所需的变量结构
            DATA: lv_params   TYPE string,
                  lt_header   TYPE if_web_http_request=>name_value_pairs,
                  ls_response TYPE send_response,
                  lv_req_json TYPE string.

            "获取Token
            DATA: lv_token TYPE string,
                  lv_match TYPE string.
            DATA: ls_loginfo TYPE send_response.
            DATA: lt_token TYPE if_web_http_request=>name_value_pairs.

            lv_params = |/sap/opu/odata/SAP/API_PROD_ORDER_CONFIRMATION_2_SRV/ProdnOrdConf2| .

            TRY.
                lo_es->get_token(
                  EXPORTING
*            ifcode      = cv_fcode01
                    lv_params   = lv_params
*         is_savelog  =
*            sendermsgid = gs_input-is_header-interfaceid
                  IMPORTING
                    lv_token    = lv_token
                    lv_match    = lv_match
*            lt_cookies = gt_cookies
                  RECEIVING
                    result      = ls_loginfo
                ).
              CATCH cx_static_check.
                ls_loginfo-status = 'E'.
                ls_loginfo-message = '获取Token异常'.
            ENDTRY.

            "获取token成功，则调用销售交货单行项目更改接口
*    IF gv_token IS INITIAL OR gv_match IS INITIAL OR gv_cookie IS INITIAL.
            IF ls_loginfo-status = 'E'.
              APPEND VALUE #(
                          %tky = ls_key-%tky
                          %msg = new_message_with_text(
                                   severity = if_abap_behv_message=>severity-error
                                   text     = ls_loginfo-message )
                        ) TO reported-zi_dt_bg_prtlabel.

              RETURN.

            ENDIF.


            " 总数量计算：箱数 × 每箱数量
            DATA(lv_calculated_yield) = ls_data-Containers * ls_data-QuantityPerContainer.
            DATA(lv_qty_str) = condense( CONV string( lv_calculated_yield ) ).

            DATA(lv_clean_order) = condense( CONV string( ls_data-ProductionOrder ) ).
            DATA(lv_clean_op)    = condense( CONV string( ls_data-ProductionOrderOperation ) ).
            DATA(lv_clean_item)  = condense( CONV string( ls_data-ProductionOrderItem ) ).
            DATA(lv_clean_prod)  = condense( CONV string( ls_data-Product ) ).
            DATA(lv_clean_plant) = condense( CONV string( ls_data-ProductionPlant ) ).
            DATA(lv_clean_loc)   = condense( CONV string( ls_data-StorageLocation ) ).
*            DATA(lv_clean_uom)   = condense( CONV string( ls_data-Meins ) ).
            DATA(lv_clean_ser)   = condense( CONV string( lv_serial ) ).

            DATA lv_commercial_uom TYPE msehi.

            SELECT SINGLE UnitOfMeasure_E
              FROM I_UnitOfMeasureText
              WHERE UnitOfMeasure = @ls_data-Meins
                AND Language      = 'E'
              INTO @lv_commercial_uom.

            " 如果万一底表没配对应的商业码，保底使用原内部技术码，确保报工流程绝不卡死
            IF sy-subrc <> 0 OR lv_commercial_uom IS INITIAL.
              lv_commercial_uom = ls_data-Meins.
            ENDIF.

            " 💡 B. 读取工序关联的 6 组基础标准工时和对应的单位参考（来源 I_ProductionOrderOperation_2）
            SELECT SINGLE WorkQuantityUnit1, WorkCenterStandardWorkQty1,
                          WorkQuantityUnit2, WorkCenterStandardWorkQty2,
                          WorkQuantityUnit3, WorkCenterStandardWorkQty3,
                          WorkQuantityUnit4, WorkCenterStandardWorkQty4,
                          WorkQuantityUnit5, WorkCenterStandardWorkQty5,
                          WorkQuantityUnit6, WorkCenterStandardWorkQty6,OperationReferenceQuantity
              FROM I_ProductionOrderOperation_2
              WHERE ProductionOrder          = @ls_data-ProductionOrder
                AND ProductionOrderOperation = @ls_data-ProductionOrderOperation
              INTO @DATA(ls_base_op_workhours).

            " C. 公式演算核心矩阵：本次报工工时 = 当前汇总总数量 * 基础工时 / 基础数量（ls_base_op_workhours-OperationReferenceQuantity）
            DATA: lv_act_work1 TYPE string VALUE '0.000',
                  lv_act_work2 TYPE string VALUE '0.000',
                  lv_act_work3 TYPE string VALUE '0.000',
                  lv_act_work4 TYPE string VALUE '0.000',
                  lv_act_work5 TYPE string VALUE '0.000',
                  lv_act_work6 TYPE string VALUE '0.000'.

            " 只有当订单总计划数量大于 0 时才执行数学分摊，防止触发被零除（Division by zero）系统运行时短转崩溃
            IF sy-subrc = 0 AND ls_base_op_workhours-OperationReferenceQuantity > 0.
              lv_act_work1 = condense( CONV string( ( lv_calculated_yield * ls_base_op_workhours-WorkCenterStandardWorkQty1 ) / ls_base_op_workhours-OperationReferenceQuantity ) ).
              lv_act_work2 = condense( CONV string( ( lv_calculated_yield * ls_base_op_workhours-WorkCenterStandardWorkQty2 ) / ls_base_op_workhours-OperationReferenceQuantity ) ).
              lv_act_work3 = condense( CONV string( ( lv_calculated_yield * ls_base_op_workhours-WorkCenterStandardWorkQty3 ) / ls_base_op_workhours-OperationReferenceQuantity ) ).
              lv_act_work4 = condense( CONV string( ( lv_calculated_yield * ls_base_op_workhours-WorkCenterStandardWorkQty4 ) / ls_base_op_workhours-OperationReferenceQuantity ) ).
              lv_act_work5 = condense( CONV string( ( lv_calculated_yield * ls_base_op_workhours-WorkCenterStandardWorkQty5 ) / ls_base_op_workhours-OperationReferenceQuantity ) ).
              lv_act_work6 = condense( CONV string( ( lv_calculated_yield * ls_base_op_workhours-WorkCenterStandardWorkQty6 ) / ls_base_op_workhours-OperationReferenceQuantity ) ).
            ENDIF.

            " 在后端根据控制特征码动态组装标准报工 JSON 载荷
            IF ls_data-OperationControlProfile <> 'ZP01'.
              lv_req_json = |\{| &&

                            |  "OrderID": "{ lv_clean_order }",| &&
                            |  "Sequence": "0",| &&
                            |  "OrderOperation": "{ lv_clean_op }",| &&
                            |  "ConfirmationYieldQuantity": "{ lv_qty_str }",| &&
                            |  "ConfirmationText": "{ ls_data-GroupUUID  }",| &&
                            |  "OpConfirmedWorkQuantity1": "{ lv_act_work1 }",| &&
                            |  "OpWorkQuantityUnit1": "{ ls_base_op_workhours-WorkQuantityUnit1 }",| &&
                            |  "OpConfirmedWorkQuantity2": "{ lv_act_work2 }",| &&
                            |  "OpWorkQuantityUnit2": "{ ls_base_op_workhours-WorkQuantityUnit2 }",| &&
                            |  "OpConfirmedWorkQuantity3": "{ lv_act_work3 }",| &&
                            |  "OpWorkQuantityUnit3": "{ ls_base_op_workhours-WorkQuantityUnit3 }",| &&
                            |  "OpConfirmedWorkQuantity4": "{ lv_act_work4 }",| &&
                            |  "OpWorkQuantityUnit4": "{ ls_base_op_workhours-WorkQuantityUnit4 }",| &&
                            |  "OpConfirmedWorkQuantity5": "{ lv_act_work5 }",| &&
                            |  "OpWorkQuantityUnit5": "{ ls_base_op_workhours-WorkQuantityUnit5 }",| &&
                            |  "OpConfirmedWorkQuantity6": "{ lv_act_work6 }",| &&
                            |  "OpWorkQuantityUnit6": "{ ls_base_op_workhours-WorkQuantityUnit6 }"| &&
                            |\}|.
            ELSE.
              lv_req_json = |\{| &&

                            |  "OrderID": "{ lv_clean_order }",| &&
                            |  "Sequence": "0",| &&
                            |  "OrderOperation": "{ lv_clean_op }",| &&
                            |  "ConfirmationYieldQuantity": "{ lv_qty_str }",| &&
                            |  "ConfirmationText": "{ ls_data-GroupUUID }",| &&
                            |  "OpConfirmedWorkQuantity1": "{ lv_act_work1 }",| &&
                            |  "OpWorkQuantityUnit1": "{ ls_base_op_workhours-WorkQuantityUnit1 }",| &&
                            |  "OpConfirmedWorkQuantity2": "{ lv_act_work2 }",| &&
                            |  "OpWorkQuantityUnit2": "{ ls_base_op_workhours-WorkQuantityUnit2 }",| &&
                            |  "OpConfirmedWorkQuantity3": "{ lv_act_work3 }",| &&
                            |  "OpWorkQuantityUnit3": "{ ls_base_op_workhours-WorkQuantityUnit3 }",| &&
                            |  "OpConfirmedWorkQuantity4": "{ lv_act_work4 }",| &&
                            |  "OpWorkQuantityUnit4": "{ ls_base_op_workhours-WorkQuantityUnit4 }",| &&
                            |  "OpConfirmedWorkQuantity5": "{ lv_act_work5 }",| &&
                            |  "OpWorkQuantityUnit5": "{ ls_base_op_workhours-WorkQuantityUnit5 }",| &&
                            |  "OpConfirmedWorkQuantity6": "{ lv_act_work6 }",| &&
                            |  "OpWorkQuantityUnit6": "{ ls_base_op_workhours-WorkQuantityUnit6 }",| &&
                            |  "to_ProdnOrdConfMatlDocItm": [| &&
                            |    \{| &&
                            |      "OrderID": "{ lv_clean_order }",| &&
                            |      "OrderItem": "{ lv_clean_item }",| &&
                            |      "Material": "{ lv_clean_prod }",| &&
                            |      "Plant": "{ lv_clean_plant }",| &&
                            |      "StorageLocation": "{ lv_clean_loc }",| &&
                            |      "GoodsMovementType": "101",| &&
                            |      "EntryUnit": "{ lv_commercial_uom }",| &&
                            |      "Batch": "{ lv_clean_ser }",| &&
                            |      "QuantityInEntryUnit": "{ lv_qty_str }",| &&
                            |      "GoodsMovementRefDocType": "F"| &&
                            |    \}| &&
                            |  ]| &&
                            |\}|.
            ENDIF.

            " 设置服务路径与 HTTP 头信息
            lv_params = |/sap/opu/odata/SAP/API_PROD_ORDER_CONFIRMATION_2_SRV/ProdnOrdConf2|.
            lt_header = VALUE #( ( name = 'x-csrf-token' value = lv_token )
                                 ( name = 'Accept'        value = 'application/json' )
                                 ( name = 'Content-Type'  value = 'application/json' ) ).

            " 发送报工请求
            ls_response = lo_es->post_data(
              EXPORTING
                lv_params    = lv_params
                lt_header    = lt_header
                request_json = lv_req_json
            ).

            " 检查响应状态
            IF ls_response-status = 'S'.

              TYPES: BEGIN OF ty_confirmation_d,
                       Confirmation        TYPE string,
                       ConfirmationCounter TYPE string,
                     END OF ty_confirmation_d,
                     BEGIN OF ty_odata_v2_wrapper,
                       d TYPE ty_confirmation_d,
                     END OF ty_odata_v2_wrapper.
              DATA ls_parsed_res TYPE ty_odata_v2_wrapper.

              /ui2/cl_json=>deserialize(
                EXPORTING
                  json        = ls_response-responsejson
                  pretty_name = /ui2/cl_json=>pretty_mode-pascal_case
                CHANGING
                  data        = ls_parsed_res
              ).

              " 提取出标准 ERP 生成的真实报工流水凭证号
              DATA(lv_conf_no)  = condense( ls_parsed_res-d-confirmation ).

              MODIFY ENTITIES OF zi_dt_bg_prtlabel IN LOCAL MODE
                ENTITY zi_dt_bg_prtlabel
                UPDATE FIELDS ( OrderOperInternalID )
                WITH VALUE #( ( %tky                = ls_key-%tky
                                OrderOperInternalID = lv_conf_no  )
                ).

              "打印改为前台控制，报工完成后不自动打印队列
**              " 报工成功 → 链式触发本类的物理打印动作
*              MODIFY ENTITIES OF zi_dt_bg_prtlabel IN LOCAL MODE
*                ENTITY zi_dt_bg_prtlabel
*                EXECUTE printLabel
*                FROM VALUE #( ( %tky = ls_key-%tky ) )
*                FAILED DATA(ls_failed_act)
*                REPORTED DATA(ls_reported_act).
*
*              IF ls_failed_act IS NOT INITIAL.
*                failed-zi_dt_bg_prtlabel =
*                  CORRESPONDING #( BASE ( failed-zi_dt_bg_prtlabel )
*                                   ls_failed_act-zi_dt_bg_prtlabel ).
*
*                reported-zi_dt_bg_prtlabel =
*                  CORRESPONDING #( BASE ( reported-zi_dt_bg_prtlabel )
*                                   ls_reported_act-zi_dt_bg_prtlabel ).
*              ENDIF.
            ELSE.

*              " 报工失败 → 不触发打印，将错误通过标准 %tky 送回前端
              DATA(lv_msg1) = CONV string( ls_response-message+0(50) ).

              DATA(lv_msg2) = CONV string(
                COND string(
                    WHEN strlen( ls_response-message ) > 50
                    THEN ls_response-message+50(50)
                    ELSE '' ) ).

              DATA(lv_msg3) = CONV string(
                COND string(
                    WHEN strlen( ls_response-message ) > 100
                    THEN ls_response-message+100(50)
                    ELSE '' ) ).

              DATA(lv_msg4) = CONV string(
                COND string(
                    WHEN strlen( ls_response-message ) > 150
                    THEN ls_response-message+150(50)
                    ELSE '' ) ).


              APPEND VALUE #(
                    %tky = ls_key-%tky
                ) TO failed-zi_dt_bg_prtlabel.

              APPEND VALUE #(
                    %tky = ls_key-%tky
                    %msg = new_message(
                        id       = 'ZPP_BG'
                        number   = '001'
                        severity = if_abap_behv_message=>severity-error
                        v1       = lv_msg1
                        v2       = lv_msg2
                        v3       = lv_msg3
                        v4       = lv_msg4 )
                ) TO reported-zi_dt_bg_prtlabel.

              RETURN.

            ENDIF.

          CATCH cx_root INTO DATA(lo_ex).
            " 抓取系统运行时发生的任何未知崩溃文本并闭环返回
            DATA(lv_crash_msg) = lo_ex->get_text( ).

            APPEND VALUE #( %tky = ls_key-%tky ) TO failed-zi_dt_bg_prtlabel.

            APPEND VALUE #(
                  %tky = ls_key-%tky
                  %msg = new_message_with_text(
                           severity = if_abap_behv_message=>severity-error
                           text     = |系统通信或内部异常: { lv_crash_msg }| )
                ) TO reported-zi_dt_bg_prtlabel.
        ENDTRY.
      ENDIF.
    ENDIF.
  ENDMETHOD.

  METHOD printLabelInFront.

    "============================================================
    " 1. Local data
    "============================================================

    DATA:
      lt_target_serials TYPE STANDARD TABLE OF zdt_bg_prtlabel-zserialno
                         WITH EMPTY KEY,

      lv_input_serials  TYPE string,

      lv_final_pdf      TYPE xstring,
      lv_base64         TYPE string,

      lv_first_sn       TYPE zdt_bg_prtlabel-zserialno.

    TRY.

        "==========================================================
        " 2. Read action parameter
        "
        " Fiori:
        " SerialNos = 'SN001,SN002,SN003'
        "
        " 一个 SN 或多个 SN 都支持
        "==========================================================

        READ TABLE keys INTO DATA(ls_key) INDEX 1.

        IF sy-subrc <> 0.
          RETURN.
        ENDIF.

        lv_input_serials = ls_key-%param-SerialNos.

*        " 如果没有传 SerialNos，则使用当前行 SerialNo
*        IF lv_input_serials IS INITIAL.
*
*          lv_input_serials =
*            CONV string( ls_key-serialno ).
*
*        ENDIF.

        IF lv_input_serials IS INITIAL.

          APPEND VALUE #(
            %tky = ls_key-%tky
            %msg = new_message_with_text(
                     severity = if_abap_behv_message=>severity-error
                     text     = '没有传入需要打印的 SerialNo。'
                   )
          ) TO reported-zi_dt_bg_prtlabel.

          APPEND VALUE #(
            %tky = ls_key-%tky
          ) TO failed-zi_dt_bg_prtlabel.

          RETURN.
        ENDIF.

        "==========================================================
        " 3. Split multiple SN
        "
        " 支持：
        " SN001
        " SN001,SN002,SN003
        "==========================================================
        DATA lt_split_strings TYPE STANDARD TABLE OF string
                              WITH EMPTY KEY.

        SPLIT lv_input_serials AT ',' INTO TABLE lt_split_strings.


        LOOP AT lt_split_strings INTO DATA(lv_split_sn).

          DATA(lv_clean_sn) =
            condense( CONV string( lv_split_sn ) ).

          IF lv_clean_sn IS NOT INITIAL.

            APPEND CONV zdt_bg_prtlabel-zserialno(
              lv_clean_sn
            ) TO lt_target_serials.

          ENDIF.
        ENDLOOP.
        " 去重
        SORT lt_target_serials.
        DELETE ADJACENT DUPLICATES FROM lt_target_serials.

        IF lt_target_serials IS INITIAL.
          APPEND VALUE #(
            %tky = ls_key-%tky
            %msg = new_message_with_text(
                     severity = if_abap_behv_message=>severity-error
                     text     = '没有有效的 SerialNo。'
                   )
          ) TO reported-zi_dt_bg_prtlabel.

          APPEND VALUE #(
            %tky = ls_key-%tky
          ) TO failed-zi_dt_bg_prtlabel.

          RETURN.
        ENDIF.

        READ TABLE lt_target_serials
          INTO lv_first_sn
          INDEX 1.
        "==========================================================
        " 4. Create PDF merger
        "
        " 一个 SN → 一个 PDF
        " 多个 SN → 多个 PDF
        " 最后统一 merge
        "==========================================================

        DATA(lo_merger) =
          cl_rspo_pdf_merger=>create_instance( ).
        "==========================================================
        " 5. Get FDP service
        "
        " 注意：
        " 这里 SerialNo 仍然是单值。
        "
        " 每一个 SN 单独调用 FDP。
        " 不需要修改 ZC_BG_PrintForm。
        " 不需要自己组 XML。
        "==========================================================

        DATA(lo_fdp_api) =
          cl_fp_fdp_services=>get_instance(
            `ZPP_SD_FORM_LABEL`
          ).


        "==========================================================
        " 6. Process every SN
        "==========================================================

        LOOP AT lt_target_serials INTO DATA(lv_serial_no).

          DATA(lt_keys) = lo_fdp_api->get_keys( ).

          READ TABLE lt_keys
            ASSIGNING FIELD-SYMBOL(<ls_serial_key>)
            WITH KEY name = 'SERIALNO'.

          IF sy-subrc <> 0.
            " error
            RETURN.
          ENDIF.

          <ls_serial_key>-value = CONV string( lv_serial_no ).

          " FDP 自动生成 XML
          DATA(lv_xml_data) =
            lo_fdp_api->read_to_xml_v2( lt_keys ).

          " Adobe Form
          DATA(lo_form_reader) =
            cl_fp_form_reader=>create_form_reader(
              'ZFORM_BG_FG_LABEL'
            ).

          DATA(lv_xdp_binary) =
            lo_form_reader->get_layout( ).

          " 每个 SN 生成一个 PDF
          DATA lv_single_pdf TYPE xstring.

          cl_fp_ads_util=>render_pdf(
            EXPORTING
              iv_xml_data   = lv_xml_data
              iv_xdp_layout = lv_xdp_binary
              iv_locale     = 'zh_CN'
            IMPORTING
              ev_pdf        = lv_single_pdf
          ).

          " 加入 PDF Merger
          lo_merger->add_document(
            lv_single_pdf
          ).

        ENDLOOP.

        "==========================================================
        " 7. Merge all PDFs
        "==========================================================

        lv_final_pdf =
          lo_merger->merge_documents( ).


        IF lv_final_pdf IS INITIAL.

          APPEND VALUE #(
            %tky = ls_key-%tky
            %msg = new_message_with_text(
                     severity = if_abap_behv_message=>severity-error
                     text     = 'PDF 合并失败，没有生成最终 PDF。'
                   )
          ) TO reported-zi_dt_bg_prtlabel.

          APPEND VALUE #(
            %tky = ls_key-%tky
          ) TO failed-zi_dt_bg_prtlabel.

          RETURN.

        ENDIF.

        "==========================================================
        " 8. PDF -> Base64
        "==========================================================
        lv_base64 =
          cl_web_http_utility=>encode_x_base64(
            lv_final_pdf
          ).

        IF lv_base64 IS INITIAL.

          APPEND VALUE #(
            %tky = ls_key-%tky
            %msg = new_message_with_text(
                     severity = if_abap_behv_message=>severity-error
                     text     = 'PDF Base64 编码失败。'
                   )
          ) TO reported-zi_dt_bg_prtlabel.

          APPEND VALUE #(
            %tky = ls_key-%tky
          ) TO failed-zi_dt_bg_prtlabel.
          RETURN.
        ENDIF.
        "==========================================================
        " 9. Return one PDF to Fiori
        "==========================================================
        APPEND VALUE #(
          %tky = ls_key-%tky

          %param = VALUE #(
            PdfBase64 =
              lv_base64

            MimeType =
              'application/pdf'

            FileName =
              |BG_LABEL_{ lv_first_sn }_{ sy-datum }_{ sy-uzeit }.pdf|
          )
        ) TO result.
        "==========================================================
        " 10. Update print history
        "
        " 每个 SN 都 +1
        "==========================================================
*        LOOP AT lt_target_serials
*          INTO DATA(lv_update_sn).
*
*          SELECT SINGLE zprint_count
*            FROM zdt_bg_prtlabel
*            WHERE zserialno = @lv_update_sn
*            INTO @DATA(lv_current_count).
*
*          IF sy-subrc <> 0.
*            CONTINUE.
*          ENDIF.
*
*          DATA(lv_next_count) =
*            lv_current_count + 1.
*
*          UPDATE zdt_bg_prtlabel
*            SET zprint_count       = @lv_next_count,
*                zlast_print_status = 'SUCCESS'
*            WHERE zserialno = @lv_update_sn.

*        ENDLOOP.
        "============================================================
        " 11. FDP error
        "============================================================

      CATCH cx_fp_fdp_error INTO DATA(lx_fdp).

        APPEND VALUE #(
          %tky = ls_key-%tky
          %msg = new_message_with_text(
                   severity = if_abap_behv_message=>severity-error
                   text =
                     |FDP 生成 XML 失败: { lx_fdp->get_text( ) }|
                 )
        ) TO reported-zi_dt_bg_prtlabel.

        APPEND VALUE #(
          %tky = ls_key-%tky
        ) TO failed-zi_dt_bg_prtlabel.
        "============================================================
        " 12. Adobe Form error
        "============================================================
      CATCH cx_fp_form_reader INTO DATA(lx_form).

        APPEND VALUE #(
          %tky = ls_key-%tky
          %msg = new_message_with_text(
                   severity = if_abap_behv_message=>severity-error
                   text =
                     |读取 Adobe Form ZFORM_BG_LABEL 失败: { lx_form->get_text( ) }|
                 )
        ) TO reported-zi_dt_bg_prtlabel.


        APPEND VALUE #(
          %tky = ls_key-%tky
        ) TO failed-zi_dt_bg_prtlabel.
        "============================================================
        " 13. ADS error
        "============================================================
      CATCH cx_fp_ads_util INTO DATA(lx_ads).
        APPEND VALUE #(
          %tky = ls_key-%tky
          %msg = new_message_with_text(
                   severity = if_abap_behv_message=>severity-error
                   text =
                     |ADS 生成 PDF 失败: { lx_ads->get_text( ) }|
                 )
        ) TO reported-zi_dt_bg_prtlabel.
        APPEND VALUE #(
          %tky = ls_key-%tky
        ) TO failed-zi_dt_bg_prtlabel.
        "============================================================
        " 14. PDF merger error
        "============================================================
      CATCH cx_rspo_pdf_merger INTO DATA(lx_merger).
        APPEND VALUE #(
          %tky = ls_key-%tky
          %msg = new_message_with_text(
                   severity = if_abap_behv_message=>severity-error
                   text =
                     |PDF 合并失败: { lx_merger->get_text( ) }|
                 )
        ) TO reported-zi_dt_bg_prtlabel.
        APPEND VALUE #(
          %tky = ls_key-%tky
        ) TO failed-zi_dt_bg_prtlabel.
        "============================================================
        " 15. Other errors
        "============================================================
      CATCH cx_root INTO DATA(lx_root).
        APPEND VALUE #(
          %tky = ls_key-%tky
          %msg = new_message_with_text(
                   severity = if_abap_behv_message=>severity-error
                   text =
                     |前台 PDF 打印失败: { lx_root->get_text( ) }|
                 )
        ) TO reported-zi_dt_bg_prtlabel.
        APPEND VALUE #(
          %tky = ls_key-%tky
        ) TO failed-zi_dt_bg_prtlabel.

    ENDTRY.

  ENDMETHOD.

  METHOD calculateFields.
    " 1. 批量读取当前正准备保存落库的【所有箱明细记录】
    READ ENTITIES OF zi_dt_bg_prtlabel IN LOCAL MODE
      ENTITY zi_dt_bg_prtlabel
      FIELDS ( ProductionOrder Product ZproductionDate )
      WITH CORRESPONDING #( keys )
      RESULT DATA(lt_new_labels).

    " 2. 因为除了SN不同其他全部一样，我们直接抓取第一条作为全局计算特征即可！
    READ TABLE lt_new_labels INTO DATA(ls_base_job) INDEX 1.
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

    DATA lv_final_batch TYPE zdt_bg_prtlabel-batch.
    CLEAR lv_final_batch.

    " 3. 规则一：物料 + 生产日期 + 工单 在历史自建表中取批次
    SELECT SINGLE Batch
      FROM zdt_bg_prtlabel
      WHERE production_order = @ls_base_job-ProductionOrder
        AND product         = @ls_base_job-Product
        AND zproduction_date = @ls_base_job-ZproductionDate
        AND batch          IS NOT INITIAL
      INTO @lv_final_batch.

    IF sy-subrc <> 0 OR lv_final_batch IS INITIAL.
      " 4. 规则二：若历史无值，则使用 物料 + 日期 获取当前最大批次 + 1
      SELECT MAX( Batch )
        FROM zdt_bg_prtlabel
        WHERE product         = @ls_base_job-Product
          AND zproduction_date = @ls_base_job-ZproductionDate
        INTO @DATA(lv_max_batch_in_db).

      IF sy-subrc = 0 AND lv_max_batch_in_db IS NOT INITIAL.
        " 提取最末尾两位的流水号数字并强行自增 + 1
        DATA lv_max_suffix(2) TYPE n.
        lv_max_suffix = lv_max_batch_in_db+6(2) + 1.

        " 提取最大批次号的前 6 位日期前缀，组合拼装出全新的连贯自增批次号
        lv_final_batch = |{ lv_max_batch_in_db+0(6) }{ lv_max_suffix }|.
      ELSE.
        " 5. 规则三：若当前物料在今天完全没有过任何批次，则直接初始化生成第一创世批次号
        " 将当前的生产日期（格式通常为 YYYY-MM-DD 或 YYYYMMDD）转换为标准的 6 位短日期字符串 YYMMDD
        lv_final_batch = |{ ls_base_job-ZproductionDate+2(6) }01|.
      ENDIF.
    ENDIF.

    " 6. 通过普通的普通的内表推导，将算好的这一个唯一批次号，直接成批赋予所有的箱明细！
    MODIFY ENTITIES OF zi_dt_bg_prtlabel IN LOCAL MODE
      ENTITY zi_dt_bg_prtlabel
      UPDATE FIELDS ( Batch )
      WITH VALUE #( FOR ls_label IN lt_new_labels
                     ( %tky  = ls_label-%tky
                       Batch = lv_final_batch ) ).
  ENDMETHOD.

ENDCLASS.


CLASS lcl_save DEFINITION INHERITING FROM cl_abap_behavior_saver.
  PROTECTED SECTION.
    METHODS save_modified REDEFINITION.
ENDCLASS.

CLASS lcl_save IMPLEMENTATION.
  METHOD save_modified.

    "------------------------------------------------------------
    " Additional Save
    "
    " RAP 主事务已经进入 Save Phase。
    "
    " 此时正式把 BGMC Process 提交执行。
    "------------------------------------------------------------

    IF lcl_buffer=>go_bgmc_process IS BOUND.

      TRY.

          lcl_buffer=>go_bgmc_process->save_for_execution( ).


        CATCH cx_bgmc INTO DATA(lx_bgmc).

          " BGMC 本身无法在这里通过普通 RAP message
          " 返回到前端，因此这里只做异常处理。
          " 如果需要持久化错误，可单独设计日志表。

        CATCH cx_root INTO DATA(lx_root).

          " 同样不要在这里再次 COMMIT / ROLLBACK。

      ENDTRY.

      CLEAR lcl_buffer=>go_bgmc_process.

    ENDIF.

  ENDMETHOD.
ENDCLASS.

