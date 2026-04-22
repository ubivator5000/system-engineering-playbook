#### Сценарии использования

##### Сценарий №2: Подписка с автоматическим ежемесячным списанием

**Предусловие:** Школа настроила подписку на курс. Студент оплатил первый месяц.

**Основной поток:**

@drawio{https://github.com/ubivator5000/system-engineering-playbook/blob/main/src/diagrams/main_flow_2.drawio}

**Исключение:**

@drawio{https://github.com/ubivator5000/system-engineering-playbook/blob/main/src/diagrams/exeption_2.drawio}

**Режим деградации (Т-Банк недоступен при рекуррентном платеже)**

@drawio{https://github.com/ubivator5000/system-engineering-playbook/blob/main/src/diagrams/degradation_mode_2.drawio}