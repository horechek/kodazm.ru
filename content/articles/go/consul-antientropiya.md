+++
date = "2015-08-07T09:12:09+03:00"
draft = false
title = "Consul. Антиэнтропия"

+++

<p>Перевод "<a href="https://www.consul.io/docs/internals/anti-entropy.html">ANTI-ENTROPY</a>" из раздела "Consul Internals".</p>

<p>Надеюсь, что хватит сил и времени на цикл статей о консуле, включая переводы некоторых разделов документации.</p>

<p>Консул использует различные методы для управления сервисами и информацией о состоянии этих сервисов. В этом разделе мы подробно рассмотрим как регистрировать сервисы и проверки(health checks), как заполняется каталог, как обновляется информация о работоспособности сервисов.</p>

<blockquote>
  <p><em>Материал для углубленного изучения</em> В этой статье раскрываются некоторые технические детали внутренностей Consul. Вам не обязательно знать все эти детали, чтобы эффективно использовать Consul. Этот материал скорее для тех, кто хочет узнать устройство без разбора исходников</p>
</blockquote>

<h3>Компоненты</h3>

<p>Очень важно понять, как компоненты <a href="https://www.consul.io/docs/internals/anti-entropy.html#agent">агент</a> и <a href="https://www.consul.io/docs/internals/anti-entropy.html#catalog">каталог</a> взаимодействуют с друг другом в плане сервисов и проверок, как они обмениваются информацией. Описание этих компонентов, приведенное ниже, позволит быстрее понять как работает антиэнтропия.</p>

<h4>Агент</h4>

<p>Каждый агент управляет собственным списком сервисов и проверок, так же как и информацией и состоянии. Агент с состоянии запускать собственные проверки и обновлять свое локальное состояние.</p>

<p>В контексте агента, проверки и сервисы имеют значительно больше различных опций. Это возможно благодаря тому, что агент может генерировать информацию о своих сервисах и их состоянии благодаря <a href="https://www.consul.io/docs/agent/checks.html">выполнению проверок</a>.</p>

<h4>Каталог</h4>

<p>Обнаружение сервисов(service discovery) опирается на каталог сервисов. Этот каталог формируется из агрегированной информации со всех агентов. Каталог - это доступ к более высокоуровневому представлению кластера, которое предоставляет информацию о доступных сервисах, их состоянии, нодах и многое другое. Все это доступно через HTTP или DNS интерфейс Consul.</p>

<p>У сервисов и проверок, в контексте каталога, значительно меньше опций, в сравнении с агентом. Каталог только записывает и возвращает информацию о сервисах, нодах и проверках.</p>

<p>Каталог поддерживается только на серверных нодах, так как он реплецируется с помощью <a href="https://www.consul.io/docs/internals/consensus.html">Raft</a>. Это необходимо для обеспечения консолидации и консистентности кластера.</p>

<h3>Антиэнтропия</h3>

<p>Энтропию можно определить как тенденцию системы к все более неупорядоченному виду. Антиэнтропия, в рамках Consul'а, это некоторый механизм, который противостоит этой тенденции для сохранения состояния кластеара в упорядоченном виде, даже при проблемах с отдельными компонентами.</p>

<p>У Consul есть четкое разделение между глобальным каталогом сервисов и локальным состоянием агента, как было описано выше. Если рассматривать антиэнтропию с учетом этих понятий, то основной механизм будет заключаться в синхронизации локального состояния агента с каталогом. Например, если пользователь зарегистрирует новый сервис или проверку с помощью агента, то агент уведомит каталог о этом новом элементе. Аналогично, если проверка удалиться через агента, то ее не станет и в каталоге.</p>

<p>Часто антиэнтропия используется для обновления зависимых данных. Когда агент запускает свои проверки, то их статус может измениться и синхронизироваться с каталогом. Используя эту информацию, каталог может более интеллектуально реагировать на запросы о его нодах и сервисах, основываясь на их доступности.</p>

<p>Во время этой синхронизации каталог проверяется на корректность. Если в каталоге есть какие-то проверки или сервисы, про которые клиент ничего не знает, то они автоматически удаляются, чтобы привести каталог к понятному для клиента виду. Consul воспринимает состояние агента как более приоритетное, если нет различий между каталогом и агентом, то будет использоваться локальная информация агнета.</p>

<h3>Периодическая синхронизация</h3>

<p>В дополнению к запуску при новом изменении в агенте, механизм антиэнтропии подразумевает долгоиграющий процесс, который периодически выполняет синхронизацию сервисов и проверок в каталоге. Это гарантирует, что каталог обладает максимально приближенным к агенту состоянием. Так же, это дает возможность обновлять каталог при потере данных или проблемах с некоторыми нодами.</p>

<p>Для оптимальной работы синхронизации, время, за которое она протекает, варьируется в зависимости от размера кластера. Таблица ниже демонстрирует зависимость между размером кластера и временем синхронизации.</p>

<table>
<thead>
<tr>
  <th>Размер кластера</th>
  <th>Время синхронизации</th>
</tr>
</thead>
<tbody>
<tr>
  <td>1 - 128</td>
  <td>1 minutes</td>
</tr>
<tr>
  <td>129 - 256</td>
  <td>2 minutes</td>
</tr>
<tr>
  <td>257 - 512</td>
  <td>3 minutes</td>
</tr>
<tr>
  <td>513 - 1024</td>
  <td>4 minutes</td>
</tr>
</tbody>
</table>

<p>Интервалы выше приблизительны. Каждый агент сам, случайным образом, выбирает момент(в рамках предельного таймаута), когда нужно выполнять синхронизацию, что бы избежать эффекта "громыхающего стада".</p>

<h3>Максимальные усилия для синхронизации</h3>

<p>Механизм антиэнтропии может сломаться по целому ряду причин, включая неправильно настроенных агентов, рабочего окружения, проблем с вводом/выводом(доступы, нехватка места на диске и т.д.), проблемы с сетью и многое другое. Из-за всего этого, агенты пытаются прикладывать максимальные усилия для синхронизации.</p>

<p>Если ошибка произойдет в момент работы механизмов антиэнтропии, то ошибка залогируется и агент продолжит работать. Механизмы антиэнтропии будут периодически запускаться, для восстановления системы после таких сбоев.</p>