.. _signals:

=======
Signals
=======

.. versionadded:: 0.5

.. note::
  
  Signals é suportado pela excelente biblioteca `blinker`_. Se você deseja ter o
  suporte aos signals disponíveis é necessário instalá-la, no entanto está 
  biblioteca não é requerida para o MongoEngine funcionar.
  

Overview
--------

Signals são encontrados dentro do módulo `mongoengine.signals`. Exceto se você
tiver especificado um sinal para receber sem qualquer argumento adicional, além
da classe emitente (o `sender`) e da instância `document`. 
Post-signals somente serão chamados se nenhuma exceção for levantada durante o 
processamento das suas funções relacionadas.

Signals disponíveis são:

`pre_init`
  Chamado durante a criação de um nova instância de :class:`~mongoengine.Document` or
  :class:`~mongoengine.EmbeddedDocument`, após os argumentos passados no construtor
  terem sido coletados, mas antes de qualquer processamento adicional ter sido feito 
  com eles (ex.: definição de valores padrões). Manipuladores para esse sinal são passados
  ao dicinário de argumentos usando os `values` dos argumentos nomeados e pode modificar
  esse dicionario antes de retorná-lo.

`post_init`
  Chamado após todo o processamento de uma nova instância de :class:`~mongoengine.Document` or
  :class:`~mongoengine.EmbeddedDocument` ter sido completada.

`pre_save`
  Chamado dentro do método :meth:`~mongoengine.document.Document.save`, mas antes
  de executrar qualquer ação.

`pre_save_post_validation`
  Chamado dentro do método :meth:`~mongoengine.document.Document.save` 
  e após a validação ter sido feita, mas antes de salvar.

`post_save`
  Chamado dentro do método :meth:`~mongoengine.document.Document.save` 
  e após todas as ações como validação, insert/update, 
  cascades, clearing dirty flag serem completadas com sucesso. Um adicional valor
  boolean é retornado como a palavra-chave `created` para indicar se a instância
  salva foi inserida (insert) ou atualizada (update).

`pre_delete`
  Chamado dentro do método :meth:`~mongoengine.document.Document.delete` e antes de operação delete.

`post_delete`
  Chamado dentro do método :meth:`~mongoengine.document.Document.delete` após a
  operação delete ter sido realizada com sucesso.

`pre_bulk_insert`
  Chamado após a validação dos documentos para um insert, mas antes de qualquer
  dado ser escrito. Neste caso, o argumento `document` é repassado por argumentos 
  `documents` que representam a lista de documents que está sendo inserido.

`post_bulk_insert`
  Chamado após uma operação em lote (bulk) ter sido realizada com sucesso. Como
  para cada `pre_bulk_insert`, o argumento `document` é omitido e substituído por
  uma lista de `documents`.
  Um argumento boolean adicional chamado `loaded` indentifica o conteúdo de `documents`
  como ambas instâncias :class:`~mongoengine.Document` quando `True` ou simplesmente uma
  lista de chaves-primárias para os registros inseridos se `False`.
  

Attaching Events
----------------

After writing a handler function like the following::

    import logging
    from datetime import datetime

    from mongoengine import *
    from mongoengine import signals

    def update_modified(sender, document):
        document.modified = datetime.utcnow()

You attach the event handler to your :class:`~mongoengine.Document` or
:class:`~mongoengine.EmbeddedDocument` subclass::

    class Record(Document):
        modified = DateTimeField()

    signals.pre_save.connect(update_modified)

While this is not the most elaborate document model, it does demonstrate the
concepts involved.  As a more complete demonstration you can also define your
handlers within your subclass::

    class Author(Document):
        name = StringField()

        @classmethod
        def pre_save(cls, sender, document, **kwargs):
            logging.debug("Pre Save: %s" % document.name)

        @classmethod
        def post_save(cls, sender, document, **kwargs):
            logging.debug("Post Save: %s" % document.name)
            if 'created' in kwargs:
                if kwargs['created']:
                    logging.debug("Created")
                else:
                    logging.debug("Updated")

    signals.pre_save.connect(Author.pre_save, sender=Author)
    signals.post_save.connect(Author.post_save, sender=Author)

Finally, you can also use this small decorator to quickly create a number of
signals and attach them to your :class:`~mongoengine.Document` or
:class:`~mongoengine.EmbeddedDocument` subclasses as class decorators::

    def handler(event):
        """Signal decorator to allow use of callback functions as class decorators."""

        def decorator(fn):
            def apply(cls):
                event.connect(fn, sender=cls)
                return cls

            fn.apply = apply
            return fn

        return decorator

Using the first example of updating a modification time the code is now much
cleaner looking while still allowing manual execution of the callback::

    @handler(signals.pre_save)
    def update_modified(sender, document):
        document.modified = datetime.utcnow()

    @update_modified.apply
    class Record(Document):
        modified = DateTimeField()


ReferenceFields and Signals
---------------------------

Currently `reverse_delete_rules` do not trigger signals on the other part of
the relationship.  If this is required you must manually handle the
reverse deletion.

.. _blinker: http://pypi.python.org/pypi/blinker
