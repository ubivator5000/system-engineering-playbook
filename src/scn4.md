#### Сценарии использования

##### Сценарий №4: Выгрузка платежей в 1С через внешнюю обработку

**Предусловие:** Школа скачала внешнюю обработку PayLect.epf из личного кабинета и настроила её в 1С.

**Основной поток:**

@drawio{https://github.com/ubivator5000/system-engineering-playbook/blob/main/src/diagrams/main_flow_4.drawio}

**Режим деградации (PayLect API недоступен для обработки)**

@drawio{https://github.com/ubivator5000/system-engineering-playbook/blob/main/src/diagrams/degradation_mode_4.drawio}