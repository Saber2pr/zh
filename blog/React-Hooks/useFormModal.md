```tsx
import { Form, Modal, ModalProps, FormProps } from 'antd';
import React, { useEffect, useState } from 'react';
import { getArray } from 'src/utils/kit';

export interface FormModal<T> {
  initValues: T;
  visible: boolean;
  onCancel: VoidFunction;
  onOk: (values: T) => any;
  forms: React.ReactNode;
  title: string;
  modalProps?: ModalProps;
  formProps?: FormProps;
  formDeps?: any[];
}

function FormModal<T>({
  visible,
  onCancel,
  onOk,
  forms,
  title,
  modalProps,
  formProps,
  formDeps,
  initValues,
}: FormModal<T>) {
  const [form] = Form.useForm(formProps?.form);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (initValues) {
      form.setFieldsValue(initValues);
    }
  }, [...getArray(formDeps), ...Object.values(initValues || {})]);

  return (
    <Modal
      {...modalProps}
      title={title}
      visible={visible}
      onCancel={onCancel}
      okButtonProps={{ ...(modalProps?.okButtonProps || {}), loading }}
      onOk={() => form.submit()}
    >
      <Form
        {...formProps}
        initialValues={initValues}
        onFinish={async values => {
          try {
            setLoading(true);
            await onOk(values);
            onCancel();
          } catch (error) {
            console.log(error);
          } finally {
            setLoading(false);
          }
        }}
        form={form}
      >
        {forms}
      </Form>
    </Modal>
  );
}

export interface UseFormModal<T> {
  initValues: T;
  forms: React.ReactNode;
  onOk: (values: T) => any;
  title: string;
  modalProps?: ModalProps;
  formProps?: FormProps;
  formDeps?: any[];
}

export function useFormModal<T>({
  forms,
  onOk,
  title,
  formDeps,
  initValues,
  ...props
}: UseFormModal<T>) {
  const [show, setShow] = useState(false);

  return {
    setShow,
    show,
    modal: (
      <FormModal
        {...props}
        title={title}
        forms={forms}
        visible={show}
        onCancel={() => setShow(false)}
        onOk={onOk}
        formDeps={formDeps}
        initValues={initValues}
      />
    ),
  };
}
```